---
title: 每天5分钟玩转Python（03） - 安装Pycharm
date: '2019-06-03 12:03:21 +0800'
toc: true
categories: [ python ]
tags: [ python教程 ]
---

安装完Python之后还需要安装集成开发环境，也就是通常所说的IDE。虽然你可以用python自带的IDLE，
或者直接使用notepad++这类本文编辑器，但是我的建议是想敲代码快一点还是用IDE吧，这里我首推Pycharm。
事实上我对Jetbrains出的编程工具系列相当偏爱，因为对比使用过后你会发现它们实在是用的太爽了。

本篇介绍如何在Windows上面安装Pycharm，以及它的一些基本使用方法。
<!-- more -->

## 下载和安装

下载地址：<https://www.jetbrains.com/pycharm/download/#section=windows>

![](https://xnstatic-1253397658.file.myqcloud.com/p03_downloadpycharm.png)

Pycharm有两个版本，一个是Professional，一个是Community版本。前者是收费的，后者是免费社区版本。
对于学习Python基础用社区版本足够了，所以我建议你下载Community版本。

下载下来后双击安装，一直点下一步就好啦，选择安装位置的时候可以选非系统盘。
安装完成后，先别急着打开Pycharm，先修改一个启动参数配置文件，因为默认配置启动会有点卡。
打开Pycharm的安装目录，进入bin目录。修改文件`pycharm64.exe.vmoptions`，将下面的内容替换进去：

```
-Xms2048m
-Xmx2048m
-XX:ReservedCodeCacheSize=240m
-XX:+UseConcMarkSweepGC
-XX:SoftRefLRUPolicyMSPerMB=50
-ea
-XX:CICompilerCount=2
-Dsun.io.useCanonPrefixCache=false
-Djava.net.preferIPv4Stack=true
-Djdk.http.auth.tunneling.disabledSchemes=""
-XX:+HeapDumpOnOutOfMemoryError
-XX:-OmitStackTraceInFastThrow
-Djdk.attach.allowAttachSelf
-Dkotlinx.coroutines.debug=off
-Djdk.module.illegalAccess.silent=true
```

保存后，再在bin目录找到`pycharm64.exe`，右键->发送到->桌面快捷方式。
然后在桌面上双击pycharm64.exe打开Pycharm。第一次会出现如下弹框，
请选择下面的`Do not import settings`

![](https://xnstatic-1253397658.file.myqcloud.com/p03_open.png)

## Pycharm设置

### 设置字体和颜色

使用快捷键`Ctl + Alt + S`，打开配置框。选择左侧的Appearance & Behavior -> Appearance，然后选择Theme，
还有主题的字体，大小。点击下面的`Apply`按钮。

![](https://xnstatic-1253397658.file.myqcloud.com/p03_theme.png)

继续选择左侧的Editor -> Font，然后在右边配置字体、大小、行高。

![](https://xnstatic-1253397658.file.myqcloud.com/p03_font.png)

还可以继续配置编辑器背景色之类，我就不去多讲了。这里面配置挺多，可以慢慢去试试。

### 编码设置

Python 的编码问题由来已久，为了避免一步一坑，Pycharm 提供了方便直接的解决方案，这里全部改成UTF-8，一统天下。

![](https://xnstatic-1253397658.file.myqcloud.com/p03_encoding.png)

在 IDE Encoding 、Project Encoding 、Property Files 三处都使用 UTF-8 编码。同时所有Python源代码文件头部都添加

```python
#!/usr/bin/env python
# -*- encoding: utf-8 -*-
```

## 创建新的工程

点击File -> New Project，创建一个新的Python工程，比如我在d:\temp\下面创建一个test工程。如下

![](https://xnstatic-1253397658.file.myqcloud.com/p03_new.png)

创建完后，在test文件上面点击右键 -> New -> Python File，填写Python文件名为main，然后写入如下内容：

```python
#!/usr/bin/env python
# -*- encoding: utf-8 -*-

print("hello world")
```

## 界面总览

现在界面应该是这样的，上面是菜单栏和工具栏，左侧是工程目录文件列表栏，中间主要部分是代码编辑区。
下面还有命令窗口、状态栏。基本上跟一般的IDE布局一致。

![](https://xnstatic-1253397658.file.myqcloud.com/p03_view.png)

## Pycharm快捷键

使用Pycharm之所以能加快我们敲代码速度就是因为它超级强大的快捷键，
熟练使用这些快捷键将极大的加快编辑速度。下面列出重要的一些快捷键和它们的用途

### 1、编辑（Editing）

    Ctrl + Space 基本的代码完成（类、方法、属性）
    Ctrl + Alt + Space 快速导入任意类
    Ctrl + Shift + Enter 语句完成
    Ctrl + P 参数信息（在方法中调用参数）
    Ctrl + Q 快速查看文档
    F1 Web帮助文档主页
    Shift + F1 选中对象的Web帮助文档
    Ctrl + 悬浮/单击鼠标左键 简介/进入代码定义
    Ctrl + Z 撤销上次操作
    Ctrl + Shift + Z 重做,恢复上次的撤销
    Ctrl + F1 显示错误描述或警告信息
    Alt + Insert 自动生成代码
    Ctrl + O 重新方法
    Ctrl + Alt + T 选中
    Ctrl + / 行注释/取消注释
    Ctrl + Shift + / 块注释
    Ctrl + W 选中增加的代码块
    Ctrl + Shift + W 回到之前状态
    Ctrl + Shift + ]/[ 选定代码块结束、开始
    Alt + Enter 快速修正
    Ctrl + Alt + L 代码格式化
    Ctrl + Alt + O 优化导入
    Ctrl + Alt + I 自动缩进
    Tab / Shift + Tab 缩进、不缩进当前行
    Ctrl+X/Shift+Delete 剪切当前行或选定的代码块到剪贴板
    Ctrl+C/Ctrl+Insert 复制当前行或选定的代码块到剪贴板
    Ctrl+V/Shift+Insert 从剪贴板粘贴
    Ctrl + Shift + V 从最近的缓冲区粘贴
    Ctrl + D 复制选定的区域或行
    Ctrl + Y 删除选定的行
    Ctrl + Shift + J 添加智能线
    Ctrl + Enter 智能线切割
    Shift + Enter 另起一行
    Ctrl + Shift + U 在选定的区域或代码块间切换
    Ctrl + Delete 删除到字符结束
    Ctrl + Backspace 删除到字符开始
    Ctrl + Numpad+/- 展开/折叠代码块（当前位置：函数、注释等）
    Ctrl + Shift + Numpad+/- 展开/折叠所有代码块
    Ctrl + F4 关闭运行的选项卡

### 2、查找/替换(Search/Replace)

    F3 下一个
    Shift + F3 前一个
    Ctrl + R 替换
    Ctrl + Shift + R 全局替换
    Ctrl + Shift + F 全局查找（可以在整个项目中查找某个字符串什么的，如查找某个函数名）
    连续敲击两次Shift键 查找函数

### 3、运行(Running)

    Alt + Shift + F10 运行模式配置
    Alt + Shift + F9 调试模式配置
    Shift + F10 运行
    Shift + F9 调试
    Ctrl + Shift + F10 运行编辑器配置
    Ctrl + Alt + R 运行manage.py任务

4、调试(Debugging)

    F8 跳过
    F7 进入
    Shift + F8 退出
    Alt + F9 运行游标
    Alt + F8 验证表达式
    Ctrl + Alt + F8 快速验证表达式
    F9 恢复程序
    Ctrl + F8 断点开关
    Ctrl + Shift + F8 查看断点

5、导航(Navigation)

    Ctrl + N 跳转到类
    Ctrl + Shift + N 跳转到符号
    Alt + Right/Left 跳转到下一个、前一个编辑的选项卡（代码文件）
    Alt + Up/Down跳转到上一个、下一个方法
    F12 回到先前的工具窗口
    Esc 从工具窗口回到编辑窗口
    Shift + Esc 隐藏运行的、最近运行的窗口
    Ctrl + Shift + F4 关闭主动运行的选项卡
    Ctrl + G 查看当前行号、字符号
    Ctrl + E 在当前文件弹出最近使用的文件列表
    Ctrl+Alt+Left/Right 后退、前进
    Ctrl+Shift+Backspace 导航到最近编辑区域（差不多就是返回上次编辑的位置）
    Alt + F1 查找当前文件或标识
    Ctrl+B / Ctrl+Click 跳转到声明
    Ctrl + Alt + B 跳转到实现
    Ctrl + Shift + I 查看快速定义
    Ctrl + Shift + B 跳转到类型声明
    Ctrl + U 跳转到父方法、父类
    Alt + Up/Down 跳转到上一个、下一个方法
    Ctrl + ]/[ 跳转到代码块结束、开始
    Ctrl + F12 弹出文件结构
    Ctrl + H 类型层次结构
    Ctrl + Shift + H 方法层次结构
    Ctrl + Alt + H 调用层次结构
    F2 / Shift + F2 下一条、前一条高亮的错误
    F4 / Ctrl + Enter 编辑资源、查看资源
    Ctrl + Shift + F11 书签助记开关
    Ctrl + #[0-9] 跳转到标识的书签
    Shift + F11 显示书签

### 6、搜索相关(Usage Search)

    Alt + F7/Ctrl + F7 文件中查询用法
    Ctrl + Shift + F7 文件中用法高亮显示
    Ctrl + Alt + F7 显示用法

### 7、重构(Refactoring)

    Alt + Delete 安全删除
    Shift + F6 重命名文件
    Ctrl + F6 更改签名
    Ctrl + Alt + N 内联
    Ctrl + Alt + M 提取方法
    Ctrl + Alt + V 提取属性
    Ctrl + Alt + F 提取字段
    Ctrl + Alt + C 提取常量
    Ctrl + Alt + P 提取参数

### 8、控制VCS/Local History

    Ctrl + K 提交项目
    Ctrl + T 更新项目
    Alt + Shift + C 查看最近的变化
    Alt + BackQuote(') VCS快速弹出

### 9、模版(Live Templates)

    Ctrl + Alt + J 当前行使用模版
    Ctrl + J 插入模版

### 10、基本(General)

    Alt + #\[0-9] 打开相应的工具窗口
    Ctrl + Alt + Y 同步
    Ctrl + Shift + F12 最大化编辑开关
    Alt + Shift + F 添加到最喜欢
    Alt + Shift + I 根据配置检查当前文件
    Ctrl + BackQuote(') 快速切换当前计划
    Ctrl + Alt + S 打开设置页
    Ctrl + Shift + A 查找编辑器里所有的动作
    Ctrl + Tab 在窗口间进行切换

Web帮助文档默认快捷键说明：

<https://www.jetbrains.com/help/pycharm/keyboard-shortcuts-and-mouse-reference.html>

最后，在Pycharm中打开Help->Keymap Reference可查看默认快捷键帮助文档：

![](https://xnstatic-1253397658.file.myqcloud.com/p03_keymap.png)

这些快捷键看上去很多，刚开始会有点生疏，但是只要你坚持使用，忘记就去查。很快就能熟能生巧自动记住，
变成一种下意识反应，这时候就是左手倚天剑，右手屠龙刀。刀剑在手，天下我有！

