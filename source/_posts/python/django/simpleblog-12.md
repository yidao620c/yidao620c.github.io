---
title: Django1.9开发博客12- i18n国际化
date: 2015-08-24 19:12:29 +0800
toc: true
categories: [ python ]
tags: [ django ]
---

国际化与本地化的目的为了能为各个不同的用户以他们最熟悉的语言和格式来显示网页。

Django能完美支持文本翻译、日期时间和数字的格式化、时区。

另外，Django还有两点优势：

1. 允许开发者和模板作者指定他们哪些app应该被翻译或被格式化为本地形式。
1. 允许用户根据自己的偏好来实现本地化显示。翻译依据语言，格式化依据国家，
   这些信息由浏览器中的`Accept-Language`头来决定。不过目前为止时区还未能实现。

参考官方文档：<https://docs.djangoproject.com/en/1.9/topics/i18n/>
<!-- more -->

### 配置

实际上django的国际化做的非常好了，配置很简单。

#### settings.py

首先在settings中，添加如下内容：

```python
from django.utils.translation import ugettext_lazy as _
LANGUAGES = (
    ('zh-cn', _('Simplified Chinese')),
    ('en', _('English')),
)
LOCALE_PATHS = (
    os.path.join(BASE_DIR, "locale"),
)
```

通过`LANGUAGES`执行语言列表，`LOCALE_PATHS`指定国际化目录。

在项目根目录下面创建一个locale文件夹，然后使用命令创建国际化文件：

```bash
django-admin.py makemessages -l zh_CN
```

执行完后，locale文件夹下面创建`zh_CN/LC_MESSAGES/django.po`，里面的内容类似下面：

```python
# SOME DESCRIPTIVE TITLE.
# Copyright (C) YEAR THE PACKAGE'S COPYRIGHT HOLDER
# This file is distributed under the same license as the PACKAGE package.
# FIRST AUTHOR <EMAIL@ADDRESS>, YEAR.
#
#, fuzzy
msgid ""
msgstr ""
"Project-Id-Version: PACKAGE VERSION\n"
"Report-Msgid-Bugs-To: \n"
"POT-Creation-Date: 2014-11-26 11:45+0800\n"
"PO-Revision-Date: YEAR-MO-DA HO:MI+ZONE\n"
"Last-Translator: FULL NAME <EMAIL@ADDRESS>\n"
"Language-Team: LANGUAGE <LL@li.org>\n"
"MIME-Version: 1.0\n"
"Content-Type: text/plain; charset=UTF-8\n"
"Content-Transfer-Encoding: 8bit\n"
"Plural-Forms: nplurals=1; plural=0;\n"

#: .\mysite\settings.py:94
msgid "Simplified Chinese"
msgstr "简体中文"

#: .\mysite\settings.py:95
msgid "English"
msgstr "English"

#: base.html
msgid "Simple Blog"
msgstr "极简博客"

msgid "Hello"
msgstr "欢迎你"

msgid "previous"
msgstr "上一页"

msgid "next"
msgstr "下一页"

```

将你页面上面需要翻译的内容写到这里面来即可。比如`previous`要翻译成`上一页`。

写好了所有的翻译后，再执行：

```bash
django-admin.py compilemessages
```

这时候会生成文件`zh_CN/LC_MESSAGES/django.mo`，这个是最终的目标文件了。

### 使用

我们用`base.html`来做演示，打开`mysite/templates/mysite/base.html`

```html
@% load staticfiles %@
@% load i18n %@
<html>
<head>
    <meta http-equiv="Content-Type" content="text/html;charset=utf-8"/>
    <title>@% trans 'Simple Blog'%@</title>
</head>
<body class="customize-support">
<div class="page-header">
    @% if user.is_authenticated %@
        <a href="@% url 'post_new' %@" class="top-menu"><span
                class="glyphicon glyphicon-plus"></span></a>
        <a href="@% url 'post_draft_list' %@" class="top-menu"><span
                class="glyphicon glyphicon-edit"></span></a>
        <p class="top-menu" style="font-size: 15pt;">@% trans 'Hello'%@ @@ user.username @@
            <small>&nbsp;</small>
            <a href="@% url 'django.contrib.auth.views.logout' %@" class="top-menu">
                <span class="glyphicon glyphicon-log-out"></span></a>
        </p>
    @% else %@
        <a href="@% url 'django.contrib.auth.views.login' %@" class="top-menu">
            <span class="glyphicon glyphicon-log-in"></span></a>
    @% endif %@
    <h1><a href="@% url 'blog.views.post_list' %@">@% trans 'Simple Blog'%@</a></h1>
</div>
...
</body>
</html>
```

注意

```html
<title>@% trans 'Simple Blog'%@</title>
```

这句，如果用户选择中文，那么就会被翻译成`极简博客`。这个在django.po文件中定义过。其他的内容也是类似，就不多说了。

好了，i18n国际化就是这么简单。

