---
title: Mkdocs 配置和使用
date: '2020-03-01 13:08:23 +0800'
toc: true
categories: [ 开发工具 ]
tags: [ 开发工具 ]
---

MkDocs是一个快速、简单、华丽的静态网站生成器，适用于构建项目文档。文档源文件以Markdown编写，并使用一个YAML文件来进行配置。
MkDocs生成完全静态的HTML网站，你可以将其部署到GitHub pages、Amzzon S3或你自己选择的其它任意地方。

* 官方地址：<https://www.mkdocs.org/>
* 中文文档地址：<https://mkdocs.zimoapps.com/>

MkDocs有一堆很好看的主题。 官方内置了两个主题：
[mkdocs](https://mkdocs.zimoapps.com/user-guide/styling-your-docs/#mkdocs)
和[readthedocs](https://mkdocs.zimoapps.com/user-guide/styling-your-docs/#readthedocs)，
也可以从[MkDocs wiki](https://github.com/mkdocs/mkdocs/wiki/MkDocs-Themes)中选择第三方主题，
或者[自定义主题](https://mkdocs.zimoapps.com/user-guide/custom-themes/)。
<!-- more -->

## 安装 Mkdocs

Mkdocs是用Python开发的工具可以使用pip命令来安装

```
pip install mkdocs
```

## 使用

使用很简单直接在命令行

```
mkdocs new my-project
```

这样就会在本地建立一个my-project文件夹 其中包括了一个mkdocs.yml和一个docs文件夹

* mkdocs.yml: 这个文件是一个配置文件主要配置你的站点名字，板块等具体配置点我
* docs: 是存放你要写的 Markdown 文档的地方初始化一个index.md文档配置点我。

在本地查看搭建的文档效果

```
$ mkdocs serve
Running at: http://127.0.0.1:8000/
```

然后访问`http://127.0.0.1:8000/`就可以看到生成文档的效果了

## github上配置文档

这个对于希望在github上展示项目文档非常非常方便，简直好用到爆炸。首先将你的github项目clone下来。
然后进入你的项目目录，先将`site/`加入`.gitignore`忽略文件中，然后执行：

```
mkdir docs
cd docs
mkdocs new .
mkdocs build
mkdocs serve
```

需要修改一下配置文件mkdocs.yml把site_name改成自己项目的名称，然后执行：

```
mkdocs gh-deploy --clean
```

这个命令会在github项目上创建一个gh-pages分支，并将当前目录中的site/目录下的内容推送到远程的gh-pages分支。
然后就可以在你访问你的文档了地址`https://{username}.github.io/{projectname}`

注意github可能需要等待几分钟才能看到页面，有一点缓存时间。

后面文档编写就放到docs目录下编写即可，每次更新文档后上传，进入docs目录，然后执行一行命令即可：

```
mkdocs gh-deploy --clean
```

## 常用配置

增加页面

```
echo 'This is about me page'　> docs/about.md
```

在配置文件mkdocs.yml中配置增加的页面

```yaml
site_name: myproject
nav:
- Home: index.md
- About: about.md
```

增加多级文档

```yaml
site_name: myproject
nav:
- Home: index.md
- Install: install.md
- Documents:
    - Usage: usage.md
    - Tutorial: tutorial.md
```

更改文档主题，修改配置文件mkdocs.yml中的theme值

```yaml
site_name: core-algorithm
site_description: 算法的python实现
copyright: Copyright © 2020 [XiongNeng](https://www.xncoding.com/)

nav:
- Home: index.md
- About: about.md
- Documents:
    - Usage: chapters/usage.md
    - Tutorial: chapters/tutorial.md

theme:
  name: mkdocs
  nav_style: light
  custom_dir: docs/resources/
  highlightjs: true
  hljs_languages:
    - yaml

markdown_extentions:
  - admonition

plugins:
  - search

extra_css:
  - resources/css/extra.css
```

## 内部子页面

注意并不需要将所有的页面都放到nav导航上面，只需要将你要显示的菜单放到导航上面。
内部页面可通过页面上的链接过去非常方便，但是要注意最好使用相对路径进行链接。

比如我想进入跟当前页面同级的某个页面叫subpage.md，则使用如下的方式，mkdoc会自己处理这个链接的实际地址：

```
[进入子页面](subpage.md)
```

## 使用主题的custom_dir

主题的custom_dir配置选项可用于指向覆盖父主题目录中的文件。父主题将是主题中name配置选项定义的主题。
custom_dir中与父主题中的文件同名的任何文件都将替换父主题中同名文件。custom_dir中的任何其他文件都将添加到父主题中。
custom_dir的内容应该镜像父主题的目录结构。您可以包含模板、JavaScript文件、CSS文件、图像、字体或主题中包含的任何其他媒体文件。

例如，mkdocs主题包含以下目录结构（部分）：

```
- css\
- fonts\
- img\
  - favicon.ico
  - grid.png
- js\
- 404.html
- base.html
- content.html
- nav-sub.html
- nav.html
- toc.html
```

要覆盖该主题中包含的任何文件，在docs目录下面新建一个resources目录，然后配置

```yaml
theme:
  name: mkdocs
  nav_style: light
  custom_dir: docs/resources/
```

## 自定义模板

内置主题在模板块中实现了许多部分，可以在main.html模板中单独覆盖。只需在custom_dir中创建一个main.html模板文件，
并在该文件中定义替换块。 只要确保main.html扩展base.html。例如，要更改MkDocs主题的标题，替换的main.html模板将包含以下内容：

```
{% extends "base.html" %}

{% block htmltitle %}
<title>Custom title goes here</title>
{% endblock %}
```

在上面的例子中，将使用自定义main.html文件中定义的htmltitle块代替父主题中定义的默认htmltitle块。
MkDocs和ReadTheDocs主题提供以下块：

* site_meta：包含文档头中的元标记。
* htmltitle：包含文档头中的页面标题。
* styles：包含样式表的链接标记。
* libs：包含页眉中包含的JavaScript库（jQuery等）。
* scripts：包含应在页面加载后执行的JavaScript脚本。
* analytics：包含分析脚本。
* extrahead：<head>中的空块，用于插入自定义标记/脚本/等。
* site_name：包含导航栏中的站点名称。
* site_nav：包含导航栏中的站点导航。
* search_box：包含导航栏中的搜索框。
* next_prev：包含导航栏中的下一个和上一个按钮。
* repo：包含导航栏中的GitHub存储库链接。
* content：包含页面的页面内容和目录。
* footer：包含页脚。

## 自定义JS脚本

将JavaScript库添加到custom_dir将使其可用，但系统不会自动将其包含在MkDocs生成的页面中。因此，需要从模板中添加链接到库。
比如你需要自定义一个my.js脚本给每个页面使用，必须通过如下方式来使用。

首先将js文件放到`docs/resources/js/my.js`，然后修改`docs/resources/main.html`模板：

```
{% extends "base.html" %}

{% block htmltitle %}
    <title>Custom title goes here</title>
{% endblock %}

{% block libs %}
    {{ super() }}
    <script src="{{ base_url }}/js/my.js"></script>
{% endblock %}
```

请注意，base_url模板变量用于确保链接始终相对于当前页面。现在生成的页面将包含指向模板提供的库的链接以及custom_dir中包含的库。

## 自定义CSS

CSS的自定义比较简单，直接通过配置即可，不需要改动模板：

```yaml
extra_css:
  - resources/css/extra.css
```

## admonition扩展

[admonition](https://python-markdown.github.io/extensions/admonition/)是个很好玩的东西，
是一个markdown文档的扩展，可通过不同颜色现实告警、备注信息，非常好用。

配置

```yaml
markdown_extentions:
  - admonition
```

使用方式，

```
!!! note
    this is a tip
```

可以使用这些：attention, caution, danger, error, hint, important, note, tip, and warning。

## material主题

这个主题更加强大一点，比如它支持中文全局搜索。

首先安装主题和一些常用插件，比如导出pdf

```
# 第一步先下载安装gtk3-runtime-3.xxx-ts-win64.exe，去下面的路径下载最新的然后安装
# https://github.com/tschoonj/GTK-for-Windows-Runtime-Environment-Installer/releases
pip install --upgrade setuptools
pip install mkdocs-material
#pip install mkdocs-pdf-export-plugin #这个插件有点晚问题，等它更新了再玩
```

注意：安装tgk3后，自动会将它的bin加入到PATH中，但是后面运行的时候回出现异常`cannot load library 'libcairo.so':`。
解决办法是将这个bin路径放到PATH的第一个位置即可。太机智了

完整配置

```yaml
site_name: 'core-algorithm'
site_author: '熊能'
site_description: '看我七十二变'
site_url: 'https://www.xncoding.com/'
copyright: Copyright © 2020 XiongNeng

# 源码地址
#repo_name: 'yidao620c/core-algorithm'
#repo_url: 'https://github.com/yidao620c/core-algorithm'
#edit_uri: 'blob/master/docs/'

nav:
- Index: index.md
- Documents:
  - 第一部分:
    - 数据结构: chapters/chapter1/post01.md
    - IO操作: chapters/chapter1/post02.md
  - 第二部分:
    - 多线程: chapters/chapter2/post03.md
    - 面向对象编程: chapters/chapter2/post04.md
    - 网络编程: chapters/chapter2/post05.md
    - 备忘录:
        - 我爱你: chapters/chapter2/temp/temp01.md
        - 买个锤子: chapters/chapter2/temp/temp02.md
- 关于我们:
  - About: about.md

#主题
theme:
  name: 'material'
  language: 'zh'  # 配置语言
  palette:  # 颜色
    primary: 'light blue'
    accent: 'indigo'
  feature:
    tabs: true  # 横向导航
  custom_dir: 'docs/resources/'

markdown_extensions:
  - admonition  # 提示块
  - footnotes  # 脚注
  - meta  # 定义元数据，通过文章上下文控制，如disqus
  - pymdownx.caret  # 下划线上标
  - pymdownx.tilde  # 删除线下标
  - pymdownx.critic  # 增加删除修改高亮注释，可修饰行内或段落
  - pymdownx.details  # 提示块可折叠
  - pymdownx.inlinehilite  # 行内代码高亮
  - pymdownx.mark  # 文本高亮
  - pymdownx.smartsymbols  # 符号转换
  - pymdownx.superfences  # 代码嵌套在列表里
  - codehilite:    # 代码高亮，显示行号
      guess_lang: false
      linenums: true
  - toc:  # 锚点
      permalink: true
#  - pymdownx.arithmatex  # 数学公式
  - pymdownx.betterem:  # 对加粗和斜体更好的检测
      smart_enable: all
#  - pymdownx.emoji:  # 表情
#      emoji_generator: !!python/name:pymdownx.emoji.to_svg
#  - pymdownx.magiclink  # 自动识别超链接
  - pymdownx.tasklist:  # 复选框checklist
      custom_checkbox: true

# PDF导出插件
plugins:
  - search
#  - pdf-export #这个插件还有点问题，没有更新

#扩展样式
extra_css:
  - resources/css/extra.css
```

这里说明一下`tabs: true # 横向导航`，material主题支持将一级横向和二级纵向导航。
默认为false表示只支持左边纵向导航，效果也不错。如果是手机小屏幕则可以选择此选项。如果文档是写给电脑大屏看的，
则可以设置为true，这时候根据nav的设置分成两级导航。注意的是对于nav中的所有一级页面全部归属为一个组。
就是刚进来的index.md这个组，如果你不设置index.md主页，则这个组就看不到了。所以一定要有一个index.md的链接。

### extra.css示例

比如我想自定义表头的背景颜色。可通过F12查看源码，把原来的css定义复制到`docs/resource/css/extra.css`下，再修改下

```css
.md-typeset table:not([class]) th {
    min-width: 5rem;
    padding: .6rem .8rem;
    background-color: #9facf6;
    color: #fff;
    vertical-align: top;
}
```

### 语法习惯

1. 使用H1做title

2. 文本修饰（带 ^^下划线^^ 的可修饰行内或段落）

* 加粗 **bold**
* 斜体字 _斜体字_
* 加粗斜体 ___粗斜体___
* 下划线 ^^Insert me^^
* 删除线 ~~Delete me~~
* 增加 {++ add ++}
* 修改 {~~ is ~> are ~~}
* 删除 {-- del --}
* 高亮 {== highlight ==}
* 注释 {>> comment <<}
* 上标 H^2^O, text^a\ superscript^
* 下标 CH~3~CH~2~OH, text~a\ subscript~
* 行内代码高亮：`:::java System.out.println("hello");` or `#!python println('hello')`
* 键盘快捷键标签：++ctrl+alt+f++

更多有趣的符号参见[PyMdown](https://facelessuser.github.io/pymdown-extensions/)

3. 注释块使用`>`

4. 代码块添加`tab="xxx"`分组，添加`hl_lines="3 4"`高亮行

代码块嵌套在列表中，==缩进4个空格==

```
- list1
    - sublist1
        code
    - sublist2
- list2
```

5. 提示块[Admonition](https://python-markdown.github.io/extensions/admonition/)

```
!!! note "custom title or blank"
    text

# 可折叠，+默认打开
???+ danger highlight blink "custom title or blank"
    text vtext text, text, v<br>
    text vtext text, text, v
    ```python
    text1 = "Hello, "
    text2 = "world!"
    print text1 + text2
```

类型说明

```
??? abstract "摘要，总结" abstract, summary, tldr
??? tip "贴士" tip, hint, important
??? note "注释，代码片段，说明" note, snippet, seealso
??? example "举例，列表" example
??? quote "引用，参考链接" quote, cite
??? info "提示，TODO" info, todo
??? warning "警告" warning, caution, attention
??? danger "危险" danger, error
??? success "成功，勾选，完成" success, check, done
??? fail "失败" failure, fail, missing
??? faq "问题，疑问，帮助" question, help, faq
??? bug "BUG" bug
```

6. 列表使用

* 一级列表使用`-`
* 二级列表使用`*`
* 三级列表使用`+`
* 子级列表缩进4个空格
* 使用复选框：`- [x] item`
* 列表内容换行：行尾2个空格。Admonition中的换行也是一样，结尾2个空格

复选框列表示例，其中`[ ]`表示不打勾，`[x]`表示打勾

```
- [x] Lorem ipsum dolor sit amet, consectetur adipiscing elit
- [x] Curabitur elit nibh, euismod et ullamcorper at, iaculis feugiat est
- [ ] Vestibulum convallis sit amet nisi a tincidunt
    - [x] In hac habitasse platea dictumst
    - [x] Sed egestas felis quis elit dapibus, ac aliquet turpis mattis
    - [ ] Praesent sed risus massa
- [x] Aenean pretium efficitur erat, donec pharetra, ligula non scelerisque
- [ ] Nulla vel eros venenatis, imperdiet enim id, faucibus nisi
```

7.表格

```
First Header | Second Header | Third Header
:----------- |:-------------:| -----------:
Left         | Center        | Right
```

## 优秀案例

有个mkdocs的非官方中文文档项目，里面的配置值得学习。地址：<https://github.com/zimocode/mkdocs-docs-zh/>


