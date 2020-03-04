---
title: Mkdocs 配置和使用
date: '2020-03-01 13:08:23 +0800'
comments: true
toc: true
categories: 开发工具
tags:
  - mkdocs
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

这样就会在本地建立一个my-project文件夹　其中包括了一个mkdocs.yml和一个docs文件夹

* mkdocs.yml: 这个文件是一个配置文件主要配置你的站点名字，板块等具体配置点我
* docs: 是存放你要写的 Markdown 文档的地方初始化一个index.md文档配置点我

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
```
site_name: myproject
nav:
- Home: index.md
- About: about.md
```

增加多级文档
```
site_name: myproject
nav:
- Home: index.md
- Install: install.md
- Documents:
    - Usage: usage.md
    - Tutorial: tutorial.md
```

更改文档主题，修改配置文件mkdocs.yml中的theme值
```
site_name: core-algorithm

nav:
- Home: index.md
- About: about.md
- Documents:
    - Usage: chapters/usage.md
    - Tutorial: chapters/tutorial.md

theme:
  name: readthedocs
  highlightjs: true
  hljs_languages:
    - yaml
  navigation_depth: 2
  titles_only: true
```


## 内部子页面

注意并不需要将所有的页面都放到nav导航上面，只需要将你要显示的菜单放到导航上面。
内部页面可通过页面上的链接过去非常方便，但是要注意最好使用相对路径进行链接。

比如我想进入跟当前页面同级的某个页面叫subpage.md，则使用如下的方式：

```
[进入子页面](../subpage/)
```


