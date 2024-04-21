---
title: Django1.9开发博客06- 模板继承
date: 2015-08-15 21:44:21 +0800
toc: true
categories: [ python ]
tags: [ django ]
---

模板继承就是网站的多个页面可以共享同一个页面布局或者是页面的某几个部分的内容。通过这种方式你就需要在每个页面复制粘贴同样的代码了。
如果你想改变页面某个公共部分，你不需要每个页面的去修改，只需要修改一个模板就行了，
这样最大化复用，减少了冗余，也减少了出错的几率，而且你敲的代码也少了。
<!-- more -->

### 创建一个base模板

一个base模板就是你全站所有页面都会继承的最基本的网站框架模板。我们在blog/templates/blog/中创建一个base.html模板：

    blog
    └───templates
        └───blog
                base.html
                post_list.html

打开base.html，然后将post_list.html的所有内容都复制过来，现在它的内容如下：

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
    <h1><a href="/">Django Girls Blog</a></h1>
</div>
<div class="content">
    <div class="row">
        <div class="col-md-8">
            @% for post in posts %@
            <div class="post">
                <h1><a href="">@@ post.title @@</a></h1>
                <p>published: @@ post.published_date @@</p>
                <p>@@ post.text|linebreaks @@</p>
            </div>
            @% endfor %@
        </div>
    </div>
</div>
</body>
</html>
```

在base.html中，将…块替换成下面的：

```html
<body>
    <div class="page-header">
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
```

我们其实就是将`@% raw %@@% for post in posts %@@% endfor %@@% endraw %@`
替换成了`@% raw %@@% block content %@@% endblock %@@% endraw %@`。
在base.html中我们创建了一个名字为content的block，其他页面可以通过继承base.html，
替换这个content块来生成新的页面，页面其他内容保持不变。

保存后，再修改post_list.html页面，只保留@% for post in posts %@@% endfor %@的内容：

```html
@% for post in posts %@
    <div class="post">
        <h1><a href="">@@ post.title @@</a></h1>
        <p>published: @@ post.published_date @@</p>
        <p>@@ post.text|linebreaks @@</p>
    </div>
@% endfor %@
```

然后添加这句到post_list.html页面的最开始部分：

```
@% extends 'blog/base.html' %@
```

这句话的意思就是该模板继承自blog/base.html模板

还有一步就是要将刚刚的内容放到@% raw %@@% block content %@和
@% endblock content %@@% endraw %@之间，这时候整个页面是这样的：

```html
@% extends 'blog/base.html' %@
@% block content %@
@% for post in posts %@
    <div class="post">
        <h1><a href="">@@ post.title @@</a></h1>
        <p>published: @@ post.published_date @@</p>
        <p>@@ post.text|linebreaks @@</p>
    </div>
@% endfor %@
@% endblock content %@
```

保存后刷新页面，看下是不是能正常工作：

![](https://xnstatic-1253397658.file.myqcloud.com/dj016.jpg)

一切都OK…
