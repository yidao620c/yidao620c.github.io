---
title: 使用pyenv创建多版本python环境
date: 2016-05-09 12:28:39 +0800
toc: true
categories: [ python ]
tags: [ python ]
---

以前一直使用`virtualenv`来管理python的包环境，但是有时候我需要多个python版本环境时候就不能愉快的玩耍了。
而[pyenv](https://github.com/pyenv/pyenv)可以帮到我。

pyenv是针对python版本的管理，通过修改环境变量的方式实现，在其官网说明上很清楚，
通过在PATH最前面插入`shims`来决定应用使用的python版本，从而将你的命令传递给正确的python程序。

我的系统环境是CentOS7.2
<!-- more -->

## 理解Shims

pyenv会在你的PATH最前面插入一个`shims`目录:

```
$(pyenv root)/shims:/usr/local/bin:/usr/bin:/bin
```

通过一个`rehashing`操作就可以在该目录匹配所有已经安装的不同版本的python命令，比如`python`, `pip`等。
所有对Python可执行文件的查找都会首先被这个shims路径截获，后面的设置就不生效了。

## 安装

推荐自动安装（保证系统上面先安装git）:

```bash
curl -L https://raw.githubusercontent.com/yyuu/pyenv-installer/master/bin/pyenv-installer | bash
```

## 环境变量

编辑`~/.bash_profile`，最后面加入:

```bash
export PYENV_ROOT="$HOME/.pyenv"
export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init -)"
eval "$(pyenv virtualenv-init -)"
export PYENV_VIRTUALENV_DISABLE_PROMPT=1
```

重启 shell:

```bash
exec $SHELL
```

## 常用命令

1. `pyenv versions` – 查看系统当前安装的python列表
2. `pyenv version`  – 查看系统当前使用的python版本
3. `pyenv install -v 3.5.3` – 安装python
4. `pyenv uninstall 2.7.13` – 卸载python
5. `pyenv rehash` – 为所有已安装的可执行文件（如：~/.pyenv/versions/bin/）创建shims，
   因此每当你增删了Python版本或带有可执行文件的包（如 pip）以后，都应该执行一次本命令）

## 版本切换

1. `pyenv global 3.5.3` – 设置全局的Python版本，通过将版本号写入`~/.pyenv/version`文件的方式
2. `pyenv local 2.7.13` – 设置面向程序的本地版本，通过将版本号写入当前目录下的`.python-version`文件的方式。
   通过这种方式设置的Python版本优先级较global高。
3. `pyenv shell 2.7.13` - 设置面向shell的Python版本，通过设置当前shell的`PYENV_VERSION`环境变量的方式

优先级: shell > local > global

## 卸载pyenv

1. 禁用pyenv很简单，只需要在`~/.bash_profile`中的`pyenv init`那行删了即可。
2. 完全移除pyenv，先执行上面第1步，然后删了pyenv的根目录: `rm -rf $(pyenv root)`

## 插件pyenv-virtualenv

官网地址: https://github.com/pyenv/pyenv-virtualenv

使用自动安装pyenv后，它会自动安装部分插件，通过`pyenv-virtualenv`插件可以很好的和`virtualenv`结合

另外，一个可选配置是在`~/.bash_profile`最后添加:

```
eval "$(pyenv virtualenv-init -)"
```

可以实现自动激活虚拟环境，这个特性非常有用建议都加上。

1. 创建虚拟环境: `pyenv virtualenv 2.7.13 virtual-env-2.7.13`，默认使用当前环境python版本。
   在文件夹`$(pyenv root)/versions/my-virtual-env-2.7.13`中创建一个基于Python 2.7.13的虚拟环境。
2. 列出虚拟环境: `pyenv virtualenvs`，对每个virtualenv显示2个, 短的只是个链接，那个`*`表示当前激活的。
3. 激活虚拟环境: `pyenv activate virtual-env-2.7.13`
4. 退出虚拟环境: `pyenv deactivate`
5. 删除虚拟环境: `pyenv uninstall virtual-env-2.7.13`

如果`eval "$(pyenv virtualenv-init -)"`写在你的shell配置中(比如上面的~/.bash_profile),
那么当`pyenv-virtualenv`进入/离开某个含有`.python-version`目录时会自动激活/退出虚拟环境。

场景使用流程:

```bash
# 先创建一个虚拟环境
pyenv versions
pyenv virtualenv 2.7.13 virtual-env-2.7.13
# 进入某个目录比如/root/work/flask-demo
pyenv local virtual-env-2.7.13
# 然后再不需要去手动激活了
```

使用pyenv来管理多版本的python命令，使用pyenv-virtualenv插件来管理多版本python包环境。爽歪歪~

