---
title: 使用GitBook编写项目文档
date: 2017-11-30 12:16:53
toc: true
categories: [ 开发工具 ]
tags: [ 开发工具 ]
---

我之前写了一篇如何在readthedoc上面发布文档的文章，现在又多了一个选择，就是使用GitBook来编写文档。
主要是觉得对于一个托管到GitHub上面的项目来讲，可以顺带的编写使用教程和文档，可以托管到`github.io`上面，
非常的方便有用。因为很多软件别人觉得好不好用，文档太重要了。
<!-- more -->

## 安装

官方文档地址：<https://toolchain.gitbook.com/>

**安装依赖软件**

**安装Nodejs**

首先安装Nodejs环境，参考我的博客《Nodejs的安装和配置》即可。

**安装gitbook-cli**

```bash
$ cnpm install gitbook-cli -g
$ gitbook -V
CLI version: 2.3.2
GitBook version: 3.2.3
```

出现上面类似的版本信息就说明安装成功了。

## 基本使用

gitbook 的基本用法非常简单，基本上就只有几个命令：

1. 使用 `gitbook init` 初始化书籍目录
2. 使用 `gitbook build` 编译书籍
3. 使用 `gitbook serve` 编译并预览书籍

首先，创建如下目录结构：

```
$ tree book/
book/
├── README.md
└── SUMMARY.md

0 directories, 2 files
```

EADME.md 和 SUMMARY.md 是两个必须文件，README.md 是对书籍的简单介绍：

```
$ cat book/README.md 
# README

This is a book powered by [GitBook](https://github.com/GitbookIO/gitbook).
```

SUMMARY.md 是书籍的目录结构。内容如下：

```
$ cat book/SUMMARY.md 
# SUMMARY

* [Chapter1](chapter1/README.md)
  * [Section1.1](chapter1/section1.1.md)
  * [Section1.2](chapter1/section1.2.md)
* [Chapter2](chapter2/README.md)
```

创建了这两个文件后，使用 `gitbook init`，它会为我们创建 `SUMMARY.md` 中的目录结构

```
$ cd book
$ gitbook init
$ tree
.
├── README.md
├── SUMMARY.md
├── chapter1
│   ├── README.md
│   ├── section1.1.md
│   └── section1.2.md
└── chapter2
    └── README.md

2 directories, 6 files
```

书籍目录结构创建完成以后，就可以使用 gitbook serve 来编译和预览书籍了：

```
[root@VM_170_150_centos book]# gitbook serve
Live reload server started on port: 35729
Press CTRL+C to quit ...

info: 7 plugins are installed 
...
info: found 5 pages 
info: found 0 asset files 
info: >> generation finished with success in 1.0s ! 

Starting server ...
Serving book on http://localhost:4000
```

`gitbook serve` 命令实际上会首先调用 `gitbook build` 编译书籍，
完成以后会打开一个 web 服务器，监听在本地的 4000 端口。

![](https://xnstatic-1253397658.file.myqcloud.com/gitbook01.png)

现在，gitbook 为我们创建了书籍目录结构后，就可以向其中添加真正的内容了，文件的编写使用 markdown 语法，
在文件修改过程中，每一次保存文件，`gitbook serve` 都会自动重新编译，所以可以持续通过浏览器来查看最新的书籍效果！

另外，你还可以下载 [gitbook 编辑器](https://www.gitbook.com/editor)，做到所见即所得的编辑。

## 个性化配置

除了修改书籍的主题外，还可以通过配置 `book.json` 文件来修改 gitbook 在编译书籍时的行为。

gitbook在编译书籍的时候会读取书籍源码顶层目录中的`book.json`，参考gitbook文档，一个典型的配置：

```json
{
    "author": "熊能 <yidao620@gmail.com>",
    "description": "scrapy简明指南",
    "generator": "site",
    "language": "zh-hans",
    "links": {
        "sidebar": {
            "我的博客": "https://www.xncoding.com"
        }
    },
    "pdf": {
        "fontSize": 12,
        "footerTemplate": null,
        "headerTemplate": null,
        "margin": {
            "bottom": 36,
            "left": 62,
            "right": 62,
            "top": 36
        },
        "pageNumbers": false,
        "paperSize": "a4"
    },
    "plugins": [ "theme-gestalt", "-theme-default", "styles-sass-fix", "multipart", "toggle-chapters" ],
    "pluginsConfig": {
        "theme-gestalt": {
            "logo": "/core-scrapy/assets/logo.png",
            "favicon": "/core-scrapy/assets/favicon.png",
            "excludeDefaultStyles": true
        },
        "disqus": {
            "shortName": "yidao620cgithubio"
        }
    },
    "styles": {
        "website": "./styles/website.scss"
    },
    "title": "Scrapy Cookbook2",
    "variables": {}
}
```

上面我添加了作者信息，简明介绍，还增加了一个博客链接。另外最重要的是后面的插件配置，关于插件我会在下面讲解。

详细的配置信息参考官方文档：<https://toolchain.gitbook.com/config.html>

## 推荐插件

gitbook还支持许多插件，用户可以从 [NPM](https://www.npmjs.com/) 上搜索 gitbook 的插件，gitbook推荐插件的命名方式为：

* gitbook-plugin-X: 插件
* gitbook-plugin-theme-X: 主题

接下来我推荐几个比较好用的插件

```bash
cnpm i -g gitbook-plugin-disqus
cnpm i -g gitbook-plugin-multipart
cnpm i -g gitbook-plugin-toggle-chapters
cnpm i -g gitbook-plugin-styles-sass-fix
cnpm i -g gitbook-plugin-multipart
cnpm i -g gitbook-plugin-toggle-chapters
cnpm i -g gitbook-plugin-theme-gestalt
cnpm i -g gitbook-plugin-theme-styles-sass-fix
cnpm i -g gitbook-plugin-styles-sass-fix
```

### theme-gestalt

这是一个主题插件，可替换掉默认主题，我使用的就是这个。还能自己定义css样式

### multipart

这个是能将一本书分成街部分，每一部分单独编号，互不影响。
对有非常多章节的书籍非常有用，分成两部分后，各个部分的章节都从 1 开始编号。

### toggle-chapters

toggle-chapters 插件的效果是：默认只在目录导航中显示章的标题，而不会显示小节的标题，
点击每一章或者每一节会显示当前章或节的子目录，如果有的话，但是同时会收起其它之前展开的章节。

### disqus

Disqus 是一个非常流行的为网站集成评论系统的工具，同样，gitbook 也可以集成 disqus 以便可以和读者交流。

首先，需要在 disqus 上注册一个账号，然后添加一个 website，这会获得一个关键字，然后在集成时配置这个关键字即可。

配置如下：

```json
{
    "plugins": ["disqus"],
    "pluginsConfig": {
        "disqus": {
            "shortName": "yidao620cgithubio"
        }
    }  
}
```

注意：上面的 `shortName` 的值就是你在 disqus 上创建的 website 获得的唯一关键字。

暂时我自己用的就这些插件，如果您有更多的建议，欢迎告诉我。

## 集成GitHub

我对GitBook最感兴趣的地方还是它能非常轻松的跟GitHub集成，对每个项目生成各自的文档或书籍，
并且可以发布至github.io上面，提供版本控制，这个对于写软件的文档来讲实在是太方便了，不得不爱它。

下面我以自己的一个实际项目来说明如果进行集成。

项目地址：<https://github.com/yidao620c/core-scrapy>

如果读者不了解 GitHub Pages 为何物，简单说就是一个可以托管静态网站的 Git 项目，
支持使用 markdown 语法以及 Jekyll 来构建，或者直接使用已经生成好的静态站点。
详细可以参考 [GitHub Pages 主页](https://pages.github.com/)

由于 gitbook 书籍可以通过 gitbook 本地构建出 site 格式，所以可以直接将构建好的书籍直接放到 GitHub Pages 中托管，
之后，可以通过如下地址访问书籍：

```
http://<username>.github.io/<project>
```

比如，我演示的这个项目的github page可以通过地址：<https://yidao620c.github.io/core-scrapy/> 访问到。

**准备工作**

你得有个`<username>.github.io`的github项目才行，没有的话去建一个。

**doc目录**

在项目根目录下面新建doc目录，然后将所有书籍源码放入到这个目录中。

**gh-pages分支**

对于你需要进行文档或书籍托管的项目，需要创建名字为`gh-pages`的分支。也就是说至少有两个分支：

* master - 保存你原来的代码，另外加一个`doc/`目录保存书籍的源码
* gh-pages - 保存书籍编译后的HTML文件

**publish脚本**

为了每次自动发布gitbook书籍到github.io上去，我自己写了个脚本：

```bash
#!/bin/bash

cd doc

# gitbook install
# install the plugins and build the static site
gitbook build

cd ..

# checkout to the gh-pages branch
git checkout gh-pages

# pull the latest updates
git pull origin gh-pages


if [[ "$?" != "0" ]]; then
    exit 1
fi
# copy the static site files into the current directory.
\cp -Rf doc/_book/* .

# remove 'node_modules' and '_book' directory
# git clean -fx gitbook/node_modules
# git clean -fx gitbook/_book
rm -rf doc/_book/

# remove website css files, except last one
ccount=`ls website-* | wc -w`
if [[ "$ccount" > 1 ]];then
    allcss=($(ls website-*))
    c=0
    for css in "${allcss[@]}"; do
       let "c=c+1"
       if [[ $c -ge $ccount ]]; then
           break;
       fi
       rm -f $css
    done
fi

# add all files
git add --all

# commit
git commit -a -m "Update docs"

# push to the origin
git push origin gh-pages

# checkout to the master branch
git checkout master
```

初次执行脚本时候注意：

将项目clone下来之后，默认只有maser分支，先手动增加`gh-pages`分支，
使用npm下载好所有的gitbook插件，并执行`gitbook install`命令。

这些做好之后再执行publish脚本，以后每次都不用管之前的步骤了。

**URL路径问题**

一开始发布之后发现logo图片、fontawesome路径都出现404错误，原因是对于每个项目而言多了个项目名称的路径，
这样就会找不到正确的URL。

解决办法：

1\. 在我的`yidao620c.github.io`项目master分支上面添加`fonts`和`css`文件夹，将相应的字体和css文件下载放入里面。

下载地址：<https://cdnjs.com/libraries/font-awesome>

fonts目录放入的字体：

* FontAwesome.otf
* fontawesome-webfont.eot
* fontawesome-webfont.svg
* fontawesome-webfont.ttf
* fontawesome-webfont.woff
* fontawesome-webfont.woff2

css目录放入的样式文件：

```
font-awesome.css
font-awesome.css.map
font-awesome.min.css
```

2\. 同时在配置文件里面，在url路径前面加入项目名前缀，比如上面的配置中：

```json
{
    "pluginsConfig": {
        "theme-gestalt": {
            "logo": "/core-scrapy/assets/logo.png",
            "favicon": "/core-scrapy/assets/favicon.png",
            "excludeDefaultStyles": true
        }
    }
}
```

可直接使用我的这个演示项目来构建您的gitbook文档或书籍。

## 生成PDF

GitBook也能生成PDF电子书，这个我暂时没有用这个功能，大家有兴趣可以去自己研究下。

生成电子书的文档地址：<https://toolchain.gitbook.com/ebook.html>

最后放一张截图：

![](https://xnstatic-1253397658.file.myqcloud.com/gitbook02.png)


