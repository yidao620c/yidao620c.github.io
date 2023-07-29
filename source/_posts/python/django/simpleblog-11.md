---
title: Django1.9开发博客11- 富文本与代码高亮
date: 2015-08-22 15:27:29 +0800
toc: true
categories: [ python ]
tags: [ django ]
---

TinyMCE是一个轻量级的基于浏览器的所见即所得编辑器，支持目前流行的各种浏览器，由JavaScript写成。
功能配置灵活简单（两行代码就可以将编辑器嵌入网页中），支持AJAX。另一特点是加载速度非常快。

django里引用TinyMCE富文本编辑器，其实很简单，前提是你知道django的静态文件配置。
其实这个我已经在前面文章提到过，可以回去再看看。

TinyMCE的官方网站是：<http://www.tinymce.com/>
<!-- more -->

下载地址：<http://download.moxiecode.com/tinymce/tinymce_4.1.9.zip>

TinyMCE的最新版本是4.1.9，下面是官网截屏：

![](https://xnstatic-1253397658.file.myqcloud.com/tinymce.png)

下载下来后，我们把它解压到工程的static/目录下面，如下图所示：

![](https://xnstatic-1253397658.file.myqcloud.com/dj101.png)

## 安装原理

安装的原理很简单，只需要在使用编辑器的页面里引用tinymce.min.js文件并初始化就可以了。
tinymce.min.js文件在tinymce项目里，
tinymce.min.js会根据初始配置里的信息找到需要用编辑器的html节点。

例如在post_edit.html页面使用编辑器，只需要在模板文件写下：

```html
@% load staticfiles %@
@% block header %@
    <link rel="stylesheet" href="@% static 'tinymce/plugins/upload/plugin.css' %@">
    <script type="text/javascript" src="@% static 'tinymce/tinymce.min.js' %@"></script>
    <script type="text/javascript">
        tinymce.init({
            selector: "textarea",
            //width: 800,
            height: 300,
            forced_root_block: false,
            plugins: [
                "advlist autolink lists link image charmap print preview anchor sh4tinymce upload",
                "searchreplace visualblocks code fullscreen",
                "insertdatetime table contextmenu paste"
            ],
            toolbar: "insertfile undo redo | styleselect | bold italic | alignleft aligncenter" +
            " alignright alignjustify | bullist numlist outdent indent | preview link image sh4tinymce"
        });
    </script>
@% endblock %@
```

这段代码的含义是 初始化 tinyMCE编辑器，selector指需要将编辑器显示在html那个标签节点，
这里选了textareas。则表示<textareas>会变成编辑器所在的位置。

另外，我还自定义一下编辑器的高度、插件、菜单项目等。具体详细配置请参考官方文档，写的都比较清楚。

## 给TinyMCE增加一个addmore插件

需求很简单，就是每次我写文章的时候需要插入某个`<!--more-->`标签，
这样可以在列表页面先只显示文章的一部分，然后碰到这个more标签就显示一个"点击阅读更多"的链接。

第一步，在tinymce/plugins文件下新增一个addmore文件夹，然后在里面新建一个plugin.min.js文件，
内容如下：

```js
tinymce.PluginManager.add("addmore", function (a) {
    a.addCommand("InsertMoreRule", function () {
        a.execCommand("mceInsertContent", !1, "[!--more--]")
    }), a.addButton("addmore", {
        icon: "addmore",
        tooltip: "Insert More",
        cmd: "InsertMoreRule"
    }), a.addMenuItem("addmore", {
        icon: "addmore",
        text: "Insert More",
        cmd: "InsertMoreRule",
        context: "insert"
    })
});
```

在post_edit.html中修改tinymce.init方法，plugins项目后面添加一个addmore：

    ...
    plugins: [
        "advlist autolink lists link image charmap print preview anchor sh4tinymce upload",
        "searchreplace visualblocks code fullscreen",
        "insertdatetime table contextmenu paste addmore"
    ],
    ...

再看看效果，没问题了。

## SyntaxHighlighter代码高亮

程序员写博客当然少不了代码高亮，这个功能页很容易实现。有一款插件叫SyntaxHighlighter值的推荐。

项目主页：<http://alexgorbatchev.com/SyntaxHighlighter/>

下载地址：<http://alexgorbatchev.com/SyntaxHighlighter/download/download.php?sh_current>

下载下来后直接解压到static/目录下面，这个跟tinymce是一样的原理。

**使用方法**

只需要修改django页面的基础模板就行了，非常简单。

打开mysite/templates/mysite/base.html页面，引入syntaxhighlighter：

```html
@% load staticfiles %@
@% load i18n %@
<html>
<head>
    <meta http-equiv="Content-Type" content="text/html;charset=utf-8"/>
    <!-- Latest compiled and minified CSS -->
    <link rel="stylesheet" href="@% static 'css/bootstrap.min.css' %@">
    <!-- Optional theme -->
    <link rel="stylesheet" href="@% static 'css/bootstrap-theme.min.css' %@">
    <!-- Blog CSS-->
    <link rel="stylesheet" href="@% static 'css/blog.css' %@">
    <link type="text/css" rel="stylesheet" href="@% static 'syntaxhighlighter/styles/shCoreDefault.css' %@"/>
    <script type="text/javascript" src="@% static 'syntaxhighlighter/scripts/shCore.js' %@"></script>
    <script type="text/javascript">SyntaxHighlighter.all();</script>
    <!-- Latest compiled and minified JavaScript -->
    <script src="@% static 'js/jquery-1.11.1.min.js' %@"></script>
    <script src="@% static 'js/base.js' %@"></script>
    <script src="@% static 'js/bootstrap.min.js' %@"></script>
    @% block header %@
    @% endblock %@
    <title>@% trans 'Simple Blog'%@</title>
</head>
<body class="customize-support">
中间省略...
<script class="javascript" src="@% static 'syntaxhighlighter/scripts/shBrushJScript.js' %@"></script>
<script class="javascript" src="@% static 'syntaxhighlighter/scripts/shBrushBash.js' %@"></script>
<script class="javascript" src="@% static 'syntaxhighlighter/scripts/shBrushPhp.js' %@"></script>
<script class="javascript" src="@% static 'syntaxhighlighter/scripts/shBrushJava.js' %@"></script>
<script class="javascript" src="@% static 'syntaxhighlighter/scripts/shBrushSql.js' %@"></script>
<script class="javascript" src="@% static 'syntaxhighlighter/scripts/shBrushXml.js' %@"></script>
<script class="javascript" src="@% static 'syntaxhighlighter/scripts/shBrushPython.js' %@"></script>
<script class="javascript" src="@% static 'syntaxhighlighter/scripts/shBrushCss.js' %@"></script>
<script class="javascript" src="@% static 'syntaxhighlighter/scripts/shBrushCpp.js' %@"></script>
</body>
</html>
```

由于我们之前已经安装过了TinyMCE，这个跟它结合起来就非常好用了，因为TinyMCE自带有选择代码语言功能。

下面是我创建文章时，插入了一段python代码的示例：

![](https://xnstatic-1253397658.file.myqcloud.com/dj102.png)

这个是保存后的效果：

![](https://xnstatic-1253397658.file.myqcloud.com/dj103.png)

## 最后一件事

别忘了部署到Heroku上面和别人分享你的成果。

OK，到此为止，前台的各种功能已经差不多了，你能一直坚持学到这里很不错了，为你自己鼓掌吧。

后面还有一个重头戏，就是django的后台管理，我选择了更美观更好用的xamdin，敬请期待...

