---
title: jinja2模板
date: 2016-09-05 22:22:22 +0800
toc: true
categories: [ python ]
tags: [ python ]
---

jinja2是python中的一个优秀的模板语言，类似于django的模板。它的速度快，安全，目前被各种框架被广泛使用。
官网地址：<http://jinja.pocoo.org/>

它的一些特性：

* 沙箱执行机制很安全
* 通过对HTML进行自动转义防止XSS攻击
* 模板继承
* 实时编译为最优化的python代码，使得它运行速度非常快
* 可选的模板预编译
* 容易调试，错误行数直接指向模板中的行
* 配置文件语法也是模板

<!-- more -->

### 安装

非常简单，一条命令搞定：

```
pip install jinja2
```

### 基本使用

通过创建一个Template类，不过这种方法不是推荐方式：

```python
from jinja2 import Template
template = Template('Hello @@ name @@!')
template.render(name='John Doe')
u'Hello John Doe!'
```

### Python3的支持

目前是支持Python3.3或以上版本的，所有单元测试都能通过，但是可能还会有隐藏的小bug。

### API

#### 基础

Jinja2使用一个中心对象叫模板环境Environment，用它来存储陪着和全局对象，
然后使用它从文件或其他地方加载模板。就算刚刚的Template例子，一个Environment对象也被隐式的创建了。

大多数框架会在程序启动时创建一个Environment对象，用它来加载模板。

最简单的陪着Jinja2和加载模板的方式类似下面：

```python
from jinja2 import Environment, PackageLoader

env = Environment(loader=PackageLoader('yourapp', 'templates'))
```

它会创建一个Environment，使用了默认配置和一个在yourapp包的templates文件夹下面搜索模板的加载器。
你还可以使用其他的加载器，或者自定义加载器，来从数据库或其他地方加载模板。

要想加载到一个模板，只需使用get_template()方法

```python
template = env.get_template('mytemplate.html')
print
template.render(the='variables', go='here')
```

#### Unicode

Jinja2内部使用Unicode，所以传给render方法的参数必须是unicode对象，另外换行符是unix的'\n'，
最好在每个python模块第一行添加

```python
# -*- coding: utf-8 -*-
```

Jinja2的模板默认编码是utf-8，一些库会严格检查str类型比如`datetime.strftime`，它不接受unicode参数。
所以Jinja2对于ascii字符串返回str，其他的返回unicode，比如：

```python
m = Template(u"@% set a, b = 'foo', 'föö' %@").module
m.a
'foo'
m.b
u'f\xf6\xf6'
```

#### 自动转换

推荐的做法是开启 Autoescape Extension扩展，下面是一个推荐的启动配置方案，对于.html,.htm和.xml的模板开启自动转换，
对于其他的禁止自动转换：

```python
def guess_autoescape(template_name):
    if template_name is None or '.' not in template_name:
        return False
    ext = template_name.rsplit('.', 1)[1]
    return ext in ('html', 'htm', 'xml')


env = Environment(autoescape=guess_autoescape,
                  loader=PackageLoader('mypackage'),
                  extensions=['jinja2.ext.autoescape'])
```

#### 加载器

加载器用来某个源比如文件来加载模板。Environment对象会将编译后的模块放到内存中，
类似于python中的sys.modules，但是不同之处在于如果模板更改了它会自动查询加载。

所有加载器都继承自BaseLoader，如果你想定义自己的加载器，就写个子类，重写它的`get_source`方法即可。
一个基本的从文件加载模板的加载器像下面这样：

```python
from jinja2 import BaseLoader, TemplateNotFound
from os.path import join, exists, getmtime


class MyLoader(BaseLoader):

    def __init__(self, path):
        self.path = path

    def get_source(self, environment, template):
        path = join(self.path, template)
        if not exists(path):
            raise TemplateNotFound(template)
        mtime = getmtime(path)
        with file(path) as f:
            source = f.read().decode('utf-8')
        return source, path, lambda: mtime == getmtime(path)
```

内置的一些加载器：

* jinja2.FileSystemLoader(searchpath, encoding='utf-8', followlinks=False)  #文件系统
* jinja2.PackageLoader(package_name, package_path='templates', encoding='utf-8')  #包
* jinja2.DictLoader(mapping)  #字典，没啥用
* jinja2.FunctionLoader(load_func) #通过一个函数来加载
* jinja2.PrefixLoader(mapping, delimiter='/') #有前缀的
* jinja2.ChoiceLoader(loaders) #多个加载器，找到第一个匹配
* jinja2.ModuleLoader(path) #预编译好的模块

#### 自定义过滤器

@% raw %@`@@ 42|myfilter(23) @@`@% endraw %@会被`myfilter(42, 23)`调用，一个例子

```python
def datetimeformat(value, format='%H:%M / %d-%m-%Y'):
    return value.strftime(format)
```

然后注册到环境中

```python
environment.filters['datetimeformat'] = datetimeformat
```

模板中使用：

```
written on: @@ article.pub_date|datetimeformat @@
publication date: @@ article.pub_date|datetimeformat('%d-%m-%Y') @@
```

还能给过滤器传入当前模板的context或environment，
所以就有了environmentfilter(), contextfilter() 和evalcontextfilter()三个装饰器。

下面是一个简单例子，将所有文本换行符转换为html换行符，文本段落转换我html段落。如果启动自动转换，返回安全的html字符串：

```python
import re
from jinja2 import evalcontextfilter, Markup, escape

_paragraph_re = re.compile(r'(?:\r\n|\r|\n){2,}')


@evalcontextfilter
def nl2br(eval_ctx, value):
    result = u'\n\n'.join(u'<p>%s</p>' % p.replace('\n', Markup('<br>\n'))
                          for p in _paragraph_re.split(escape(value)))
    if eval_ctx.autoescape:
        result = Markup(result)
    return result
```

### 模板语法

介绍完API后，开始介绍模板设计语法

#### 基础

下面是一个典型的模板：

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <title>My Webpage</title>
</head>
<body>
<ul id="navigation">
    @% for item in navigation %@
    <li><a href="@@ item.href @@">@@ item.caption @@</a></li>
    @% endfor %@
</ul>

<h1>My Webpage</h1>
@@ a_variable @@

{# a comment #}
</body>
</html>
```

默认的语法配置是这样的，你可以自己自定义：

    * @% raw %@@% ... %@@% endraw %@ 这个是语句
    * @% raw %@@@ ... @@@% endraw %@ 这个是表达式，打印模板输出
    * @% raw %@{# ... #}@% endraw %@ 这个是注释部分，不会输出
    * #  ... #  这个是行语句

#### 变量

两种访问方式：

```
@@ foo.bar @@
@@ foo['bar'] @@

# 过滤器
@@ name|striptags|title @@

# 测试
@% if loop.index is divisibleby(3) %@
```

空行控制，如果trim_blocks 和 lstrip_blocks都启用，那么模板自己的控制块行将不输出。

#### 行语句

下面等价

```html

<ul>
    # for item in seq
    <li>@@ item @@</li>
    # endfor
</ul>

<ul>
    @% for item in seq %@
    <li>@@ item @@</li>
    @% endfor %@
</ul>
```

#### 模板继承

一个基础模板

```html
<!DOCTYPE html>
<html lang="en">
<head>
    @% block head %@
    <link rel="stylesheet" href="style.css"/>
    <title>@% block title %@@% endblock %@ - My Webpage</title>
    @% endblock %@
</head>
<body>
<div id="content">@% block content %@@% endblock %@</div>
<div id="footer">
    @% block footer %@
    &copy; Copyright 2008 by <a href="http://domain.invalid/">you</a>.
    @% endblock %@
</div>
</body>
</html>
```

一个子模板继承上面的基础模板

```html
@% extends "base.html" %@
@% block title %@Index@% endblock %@
@% block head %@
@@ super() @@
<style type="text/css">
    .important {
        color: #336699;
    }
</style>
@% endblock %@
@% block content %@
<h1>Index</h1>
<p class="important">
    Welcome to my awesome homepage.
</p>
@% endblock %@
```

还能指定模板名字，更加具有可读性

```html
@% block sidebar %@
@% block inner_sidebar %@
...
@% endblock inner_sidebar %@
@% endblock sidebar %@
```

#### 字符转换

可自动转换html字符，也可以手动转换

```
@@ user.username|e @@
```

当使用自动转换时，所有的会被转换，除非你声明它是安全的。有两种方式声明安全：

* context字典中指定MarkupSafe.Markup,
* 模板使用 |safe过滤器

#### 流程控制语句

列表循环

```html
<h1>Members</h1>
<ul>
    @% for user in users %@
    <li>@@ user.username|e @@</li>
    @% endfor %@
</ul>

@% for user in users %@
@%- if loop.index is even %@@% continue %@@% endif %@
...
@% endfor %@
```

key-value形式的循环：

```html

<dl>
    @% for key, value in my_dict.iteritems() %@
    <dt>@@ key|e @@</dt>
    <dd>@@ value|e @@</dd>
    @% endfor %@
</dl>
```

if判断

```html
@% if kenny.sick %@
Kenny is sick.
@% elif kenny.dead %@
You killed Kenny!  You bastard!!!
@% else %@
Kenny looks okay --- so far
@% endif %@
```

赋值语句

```
@% set navigation = [('index.html', 'Index'), ('about.html', 'About')] %@
@% set key, value = call_something() %@
```

块赋值

```
@% set navigation %@
    <li><a href="/">Index</a>
    <li><a href="/downloads">Downloads</a>
@% endset %@
```

包含模板

```
@% include 'header.html' %@
@% include "sidebar.html" ignore missing %@
@% include "sidebar.html" ignore missing with context %@
@% include "sidebar.html" ignore missing without context %@
    Body
@% include 'footer.html' %@
```

#### i18n

国际化例子

```
<p>@% trans user=user.username %@Hello @@ user @@!@% endtrans %@</p>
@@ _('Hello %(user)s!')|format(user=user.username) @@
```

临时指定autoescape

```
@% autoescape true %@
    Autoescaping is active within this block
@% endautoescape %@

@% autoescape false %@
    Autoescaping is inactive within this block
@% endautoescape %@
```

### jinja2扩展

添加扩展

```python
jinja_env = Environment(extensions=['jinja2.ext.i18n'])
```

### FAQ

参考 <http://jinja.pocoo.org/docs/dev/faq/>

