---
title: Django1.9开发博客05- 页面美化
date: 2015-08-12 10:23:50 +0800
toc: true
categories: [ python ]
tags: [ django ]
---

css是一种用来描述某种标记语言写的web站点的样式语言。这里我们并不想展开讨论，
关于CSS我在这里推荐一个很不错的资源： [Codeacademy HTML & CSS course][]

不想从头开始写，因为我们有现成的css框架，没必要重复造轮子。
<!-- more -->

### 使用Bootstrap

目前最流行的css框架非bootstrap莫属了，官网地址：<http://getbootstrap.com/>

只需要在你的html模板页面的开始部分添加下面几句就行了

```html

<link rel="stylesheet" href="//maxcdn.bootstrapcdn.com/bootstrap/3.2.0/css/bootstrap.min.css">
<link rel="stylesheet" href="//maxcdn.bootstrapcdn.com/bootstrap/3.2.0/css/bootstrap-theme.min.css">
<script src="//maxcdn.bootstrapcdn.com/bootstrap/3.2.0/js/bootstrap.min.js"></script>
```

你的工程里面不需要引入任何的文件，因为这里直接引用了bootstrap公共的css和js文件。

再次打开模板文件，效果如下：

![](https://xnstatic-1253397658.file.myqcloud.com/dj011.jpg)

是不是感觉变美观了。^_^

### django静态文件

这里我还将讲解下django中的静态文件。静态文件就是css、js、图片、视频等等那些内容不会改变的文件，不管任何时候，对于任何用户都是一样的。

css就是一种静态文件，为了自定义css，我们必须先再django中配置，你只需要配置一次就可以了。那让我们马上开始吧！

### django中配置静态文件

首先我们需要创建一个目录来存储静态文件，在manage.py的同级目录中创建一个static文件夹

```
mysite
├─── static
└─── manage.py
```

打开配置文件mysite/settings.py，在最后面添加如下配置：

```python
STATICFILES_DIRS = (
    os.path.join(BASE_DIR, "static"),
)
```

它告知django应该在哪个位置去查找静态文件。

### 第一个CSS文件

现在我们开始创建自己的css文件了，首先在static目录中新建一个css目录，然后在里面创建一个blog.css文件。目录结构如下

```
static
└─── css
     └───blog.css
```

打开文件static/css/blog.css后，添加如下内容

```css
h1 a {
    color: #FCA205;
}
```

h1 a是CSS选择器，上面的意思是在h1标签下的链接a的文字颜色会是#FCA205，其实就是橘黄色，颜色都是用十六进制表示的。

接下来我们要让模板加载静态css文件，打开blog/templates/blog/post_list.html，在最开始部分加入：

```
@% load staticfiles %@
```

然后在bootstrap引用的后面添加下面这句

```html

<link rel="stylesheet" href="@% static 'css/blog.css' %@">
```

最后，整个模板文件类似这样：

```html
@% load staticfiles %@
<html>
<head>
    <title>Django Girls blog</title>
    <link rel="stylesheet" href="//maxcdn.bootstrapcdn.com/bootstrap/3.2.0/css/bootstrap.min.css">
    <link rel="stylesheet" href="//maxcdn.bootstrapcdn.com/bootstrap/3.2.0/css/bootstrap-theme.min.css">
    <script src="//maxcdn.bootstrapcdn.com/bootstrap/3.2.0/js/bootstrap.min.js"></script>
    <link rel="stylesheet" href="@% static 'css/blog.css' %@">
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

OK，保存并刷新后看看效果

![](https://xnstatic-1253397658.file.myqcloud.com/dj012.jpg)

我想要文字左边的边距大一点，这样会好看些。那么在blog.css中添加如下内容：

```css
body {
    padding-left: 15px;
}
```

刷新页面后效果：

![](https://xnstatic-1253397658.file.myqcloud.com/dj013.jpg)

我还想自定义文字标题的字体，在post_list.html模板的中添加如下一句

```html

<link href="http://fonts.googleapis.com/css?family=Lobster&subset=latin,latin-ext" rel="stylesheet" type="text/css">
```

这句会引入Google的一个字体Lobster，然后修改blog.css中的h1 a的样式如下：

```css
h1 a {
    color: #FCA205;
    font-family: 'Lobster';
}
```

刷新后的效果：

![](https://xnstatic-1253397658.file.myqcloud.com/dj014.jpg)

### CSS中的class

在CSS中有一个class的概念，它可以让你只改变HTML中某一部分的样式而不会影响到其他部分。

这里我们将区别标题头和文章本身的样式。

在post_list.html中添加如下的标题段：

```html
<div class="page-header">
    <h1><a href="/">Django Girls Blog</a></h1>
</div>
```

文章列表段修改如下：

```html
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
```

blog.css样式修改如下：

```css
.page-header {
    background-color: #ff9400;
    margin-top: 0;
    padding: 20px 20px 20px 40px;
}
.page-header h1, .page-header h1 a, .page-header h1 a:visited, .page-header h1 a:active {
    color: #ffffff;
    font-size: 36pt;
    text-decoration: none;
}
.content {
    margin-left: 40px;
}
h1, h2, h3, h4 {
    font-family: 'Lobster', cursive;
}
.date {
    float: right;
    color: #828282;
}
.save {
    float: right;
}
.post-form textarea, .post-form input {
    width: 100%;
}
.top-menu, .top-menu:hover, .top-menu:visited {
    color: #ffffff;
    float: right;
    font-size: 26pt;
    margin-right: 20px;
}
.post {
    margin-bottom: 70px;
}
.post h1 a, .post h1 a:visited {
    color: #000000;
}
```

保存这些文件后，刷新页面，看看，是不是很酷了：

![](https://xnstatic-1253397658.file.myqcloud.com/dj015.jpg)

已经比较美观了。上面的css应该看起来不会那么难，可以自己试着去修改它，没关系的，反正出错了可以撤销。最好多练习才行。

下一节我将介绍django中模板的继承机制！

[Codeacademy HTML & CSS course]: http://www.codecademy.com/tracks/web
