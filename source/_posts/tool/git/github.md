---
title: GitHub的一些好玩的技巧
date: 2017-02-09 09:18:01 +0800
toc: true
categories: [ 开发工具 ]
tags: [ git, github ]
---

GitHub用了这么久才发现其实上面也可以做很多好玩的东西，让我可以更加喜欢它了。
这里我总结自己知道的，肯定还有一些我还不知道的，以后看到了就补充上去。

拖拽代码到Gist，打开<https://gist.github.com/>，然后直接把本地源文件拖过去，它里面的代码就移过去了。
<!-- more -->

## 代码链接

有时候我们给别人别想某个代码文件的某段代码的时候，可能还要让别人自己在文件里对着行数找，
但是这个小技巧可以方便很多，只要在你的代码的URL文件后面添加`#Lxx`其中 xx 就是行数，
可以是单独的一行，也可以是某一段，比如 #L3-L9 表示的就是该代码里的第3到第9行。

比如我要分享自己一段代码: <https://github.com/yidao620c/simpleblog/blob/master/blog/views.py#L21-L27>

试下点击看看，跳转到代码源文件并高亮了这一段。

## 评论和issue

评论支持markdown语法，里面可以引用别人的评论，格式:

```md
> other comments

reply to
```

评论中还能插入图片，使用markdown的图片语法或者更强大的直接截图后Ctrl+V粘贴。

issue中输入冒号 : 添加表情

## 任务列表

Issues 和 Pull requests 里可以添加复选框，语法如下（注意空白符）:

```
- [ ] Be awesome
- [ ] Prepare dinner
  - [ ] Research recipe
  - [ ] Buy ingredients
  - [ ] Cook recipe
- [ ] Sleep
```

普通的markdown文件中可创建只读的任务列表，比如在README.md中添加 TODO list:

```
- [ ] Mercury
- [x] Venus
- [x] Earth
  - [x] Moon
- [x] Mars
  - [ ] Deimos
  - [ ] Phobos
```

## emoji图标

在GitHub上面很多地方都可以使用emoji图标，比如:

* issues
* markdown files
* pull requests
* commit messages
* and almost everywhere you find a textfield

一个例子:

```
My :turtle: :heart: :watermelon:
```

GitHub可使用的emoji表情: https://github.com/leereilly/emoji

Emoji 速查表: http://www.emoji-cheat-sheet.com

## 键盘快捷键

这个就更强大了，在仓库页面上提供了快捷键方便快速导航

* 按 t 键打开一个文件浏览器，这个最常用。
* 按 w 键打开分支选择菜单。
* 按 s 键聚焦光标到当前仓库的搜索框。此时按退格键就会从搜索当前仓库切换到搜索整个Github网站。
* 按 ? 查看当前页面支持的快捷键列表

## 嵌入GitHub

是否想在其他网页上面嵌入你自己的GitHub仓库页面，有个star或fork按钮。可以这样写:

```html

<iframe src="https://ghbtns.com/github-btn.html?user=yidao620c&amp;repo=python3-cookbook&amp;type=watch&amp;count=true&amp;size=large"
        allowtransparency="true" frameborder="0" scrolling="0" width="156px" height="30px"></iframe>
<iframe src="https://ghbtns.com/github-btn.html?user=yidao620c&amp;repo=python3-cookbook&amp;type=fork&amp;count=true&amp;size=large"
        allowtransparency="true" frameborder="0" scrolling="0" width="156px" height="30px"></iframe>
```

把user和repo改成你自己的就可以了，效果如下:

<iframe src="https://ghbtns.com/github-btn.html?user=yidao620c&amp;repo=python3-cookbook&amp;type=watch&amp;count=true&amp;size=large" allowtransparency="true" frameborder="0" scrolling="0" width="156px" height="30px"></iframe>
<iframe src="https://ghbtns.com/github-btn.html?user=yidao620c&amp;repo=python3-cookbook&amp;type=fork&amp;count=true&amp;size=large" allowtransparency="true" frameborder="0" scrolling="0" width="156px" height="30px"></iframe>

## 设置项目语言

GitHub会根据相关文件代码的数量来自动识别你这个项目，如果你需要自己指定项目语言，可以在项目的根目录下添加如下`.gitattributes`
文件，写入:

```
*.html linguist-language=Java
```

主要意思是把所有 html 文件后缀的代码识别成Java文件。


