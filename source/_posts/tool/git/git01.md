---
title: git简明教程 - 基础篇
date: 2017-02-06 21:32:30 +0800
toc: true
categories: [ 开发工具 ]
tags: [ git ]
---

很早就想些一篇关于git的文章了，这玩意儿实在好用，但是内容又比较多，
这里我讲解最基本使用技巧，这个足以应对99%以上的场景，剩下那些真的要用到就去看官网手册。

Git是目前世界上最先进的分布式版本控制系统（没有之一），它的诞生也是个很有趣的故事。
大家都知道Git是Linus大神写的，据说刚开始的时候，linux内核源码使用BitKeeper这个商业版本控制系统，
BitKeeper授权Linux社区免费使用，但是某一天开发Samba的Andrew这个家伙试图破解BitKeeper协议，东窗事发。
于是BitKeeper公司一怒之下收回了免费使用权。Linus大神是不可能去道歉的，于是他就花了2个星期用C语言写了Git，
一个月内，Linux源码就由Git管理了，无敌是多么寂寞 →_→
<!-- more -->

相比较像svn这样的集中式版本管理，分布式版本管理优势在哪里呢？
这里先说两个，后面再说另外几个杀手级优点。

首先，分布式所有客户机都有一个完整拷贝，所以不用担心服务器挂点。
另外分布式不需要联网就可以工作，没有中央服务器。

## 安装git

默认的yum源中都是旧的1.8版本，使用下面方法安装最新的git2版本:

```bash
sudo yum remove git
sudo yum install epel-release
sudo yum install https://centos7.iuscommunity.org/ius-release.rpm
sudo yum install git2u
```

如果在windows上面，就去官网下载安装文件，点击安装即可。

安装完成，还要简单配置下全局设置:

```bash
git config --global user.name "Your Name"
git config --global user.email "email@example.com"
# 彩色的 git 输出：
git config --global color.ui true
# 显示历史记录时，每个提交的信息只显示一行：
git config --global format.pretty oneline
```

## 初始化

### 初始化仓库:

```bash
mkdir gitdemo && cd gitdemo
git init
# Initialized empty Git repository in /root/gitdemo/.git/
```

当前目录下多了一个.git的目录，这个目录是Git来跟踪管理版本库的，
没事千万不要手动修改这个目录里面的文件，不然改乱了，就把Git仓库给破坏了。

### .gitignore文件

如果你有一些文件不需要版本跟踪就写到这里面，比如:

```
build/
*.pyc
```

接受通配符，具体规则请参考[gitignore说明](https://git-scm.com/docs/gitignore)

### 添加文件到版本库

先编写一个readme.txt文件，内容如下:

```
hello git
I like git very much
```

第一步，使用`git add`命令添加read.txt到git:

```bash
git add readme.txt
```

执行上面的命令，没有任何显示，这就对了

第二步，使用`git commit`命令提交到git仓库:

```bash
git commit -m "readme file"
```

输出:

```
[master (root-commit) 50f5fdc] readme file
 1 file changed, 2 insertions(+)
 create mode 100644 readme.txt
```

## 回退和撤销

刚刚提交完后，继续编辑readme.txt，内容如下:

```
hello git
I like git very much
add something
```

现在，运行`git status`命令看看结果：

```
[root@controller161 gitdemo]# git status
On branch master
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)

	modified:   readme.txt

no changes added to commit (use "git add" and/or "git commit -a")
```

`git status`命令可以让我们时刻掌握仓库当前的状态，上面的命令告诉我们，readme.txt被修改过了，但还没有准备提交的修改。

虽然Git告诉我们readme.txt被修改了，但如果能看看具体修改了什么内容，自然是很好的，这时候使用`git diff`命令:

```
[root@controller161 gitdemo]# git diff readme.txt
diff --git a/readme.txt b/readme.txt
index 2236b09..4114c7f 100644
--- a/readme.txt
+++ b/readme.txt
@@ -1,2 +1,3 @@
 hello git
 I like git very much
+add something
```

知道了对readme.txt作了什么修改后，再把它提交到仓库就放心多了，步骤还是先add，再commit:

```bash
git add readme.txt
```

执行add之后，我们先不提交，看看状态:

```
[root@controller161 gitdemo]# git status
On branch master
Changes to be committed:
  (use "git reset HEAD <file>..." to unstage)

	modified:   readme.txt
```

`git status`告诉我们，将要被提交的修改包括readme.txt，下一步，就可以放心地提交了:

```bash
[root@controller161 gitdemo]# git commit -m "modify readme"
[master 1d57c05] modify readme
 1 file changed, 1 insertion(+)
```

提交后，我们再用git status命令看看仓库的当前状态：

```bash
[root@controller161 gitdemo]# git status
On branch master
nothing to commit, working directory clean
```

没有需要提交的修改，而且，工作目录是干净（working directory clean）的

### 版本回退

现在再修改readme.txt文件如下:

```
hello git
I like git very much
add something
我又加了点东西在后面
```

然后再提交:

```
[root@controller161 gitdemo]# git add readme.txt
[root@controller161 gitdemo]# git commit -m "添加中文行"
[master 90507c6] 添加中文行
 1 file changed, 1 insertion(+)
```

像这样，你不断对文件进行修改，然后不断提交修改到版本库里，实际上这个commit操作就相对于一个快照。
以后你误改误删了文件是可以回退的。

我们可以通过`git log`命令查看提交历史:

```
[root@controller161 gitdemo]# git log --pretty=oneline
90507c6859f8af73a651f033de8ea901811cb4e8 添加中文行
1d57c05ee44d590f279050276884551e77d2ffb1 modify readme
50f5fdcea453fed2eee690b5e686994040ffe210 readme file
```

第一列是commit的一个id号(版本号)，是SHA1计算出来的一个非常大的数字，用十六进制表示。

或者你想通过 ASCII 艺术的树形结构来展示所有的分支, 每个分支都标示了他的名字和标签:

```bash
git log --graph --oneline --decorate --all
```

看看哪些文件改变了:

```
git log --name-status
```

假如你想回退到`modify readme`那个版本，可以通过`git reset`命令:

```
[root@controller161 gitdemo]# git reset --hard 1d57c05ee44
Unstaged changes after reset:
M	readme.txt
```

reset后面指定版本号，你可以只取前面几位，只要能区分就行。
我们再来看readme.txt:

```
[root@controller161 gitdemo]# cat readme.txt
hello git
I like git very much
add something
```

确实回退到那个提交版本了。

现在，你回退到了某个版本，关掉了电脑，第二天早上就后悔了，想恢复到新版本怎么办？找不到新版本的commit id怎么办？

在Git中，总是有后悔药可以吃的。回退必须指定commit id，而Git提供了一个命令git reflog用来记录你的每一次命令：

```
[root@controller161 gitdemo]# git reflog
1d57c05 HEAD@{0}: reset: moving to 1d57c05ee44
90507c6 HEAD@{1}: commit: 添加中文行
1d57c05 HEAD@{2}: commit: modify readme
50f5fdc HEAD@{3}: commit (initial): readme file
```

## 利用rebase合并多个commit

在使用Git作为版本控制的时候，我们可能会由于各种各样的原因提交了许多临时的 commit，
而这些 commit 拼接起来才是完整的任务。那么我们为了避免太多的 commit 而造成版本控制的混乱，
通常我们推荐将这些 commit 合并成一个

首先假设我们有如下几个 commit:

```
[root@controller161 gitdemo]# git log --pretty=oneline --abbrev-commit
ad6fa66 ok , I resolve confict
3c18f63 modify README again
10c9f30 modify readme by xiao ming
796e584 dev branch modify readme by self
dbf4ee5 Initial commit
```

我想将最近3个（ad6fa66、3c18f63、10c9f30）合并为1个提交历史，怎样做呢？

```bash
git rebase -i 796e584
```

注意这个796e584是指的合并提交的前一个，不参与合并。出现下面的编辑界面:

```
pick 3c18f63 modify README again
pick 10c9f30 modify readme by xiao ming

# Rebase 796e584..ad6fa66 onto 796e584 (2 command(s))
#
# Commands:
# p, pick = use commit
# r, reword = use commit, but edit the commit message
# e, edit = use commit, but stop for amending
# s, squash = use commit, but meld into previous commit
# f, fixup = like "squash", but discard this commit's log message
# x, exec = run command (the rest of the line) using shell
# d, drop = remove commit
```

很奇怪的是第一条commit老不显示，应该是解决冲突的commit根本没办法合并原因。

可以看到其中分为两个部分，上方未注释的部分是填写要执行的指令，而下方注释的部分则是指令的提示说明。
指令部分中由前方的命令名称、commit hash 和 commit message 组成。

当前我们只要知道 pick 和 squash 这两个命令即可。

1. pick 的意思是要会执行这个 commit
2. squash 的意思是这个 commit 会被合并到前一个commit

我们将 10c9f30 这个 commit 前方的命令改成 squash 或 s，然后输入:wq以保存并退出。

Git会压缩提交历史，如果有冲突，需要修改，修改的时候要注意，保留最新的历史，
不然我们的修改就丢弃了。修改以后要记得敲下面的命令：

```bash
git add .
git rebase --continue
```

如果你想放弃这次压缩的话，执行以下命令：

```bash
git rebase --abort
```

如果没有冲突，或者冲突已经解决，则会出现如下的编辑窗口，出现下面的界面:

```
# This is a combination of 2 commits.
# The first commit's message is:

modify README again

# This is the 2nd commit message:

modify readme by xiao ming
```

然后修改成下面commit说明的:

```
modify README again , modify readme by xiao ming
```

输入wq保存并退出, 再次输入git log查看 commit 历史信息，你会发现这3个 commit 已经合并了。

```
[root@controller161 gitdemo]# git log --pretty=oneline --abbrev-commit
4d07b18 modify README again , modify readme by xiao ming
796e584 dev branch modify readme by self
dbf4ee5 Initial commit
```

很奇怪，之前这个`ad6fa66 ok , I resolve confict`也被合并了，好神秘哦。有明白为什么的同学告我下。

## 两个分支rebase

rebase还有一种用法，就是当两个分支产生分叉，比如master和dev，
最后你想让dev分支历史看起来像没有经过任何合并一样，你也许可以用 git rebase:

```bash
$ git checkout dev
$ git rebase master
```

这些命令会把你的dev分支里的每个提交(commit)取消掉，
并且把它们临时保存为补丁(patch)(这些补丁放到".git/rebase"目录中),
然后把dev分支更新到最新的master分支，最后把保存的这些补丁应用到master分支上。

当dev分支更新之后，它会指向这些新创建的提交(commit),而那些老的提交会被丢弃。
如果运行垃圾收集命令(pruning garbage collection), 这些被丢弃的提交就会删除

同样和合并提交一样，在rebase的过程中，也许会出现冲突(conflict)，在这种情况，Git会停止rebase并会让你去解决冲突；
在解决完冲突后，用`git add`命令去更新这些内容的索引(index), 然后，你无需执行`git commit`,只要执行:

```bash
$ git rebase --continue
```

这样git会继续应用(apply)余下的补丁，
同样，在任何时候，你可以用--abort参数来终止rebase的行动，并且dev分支会回到rebase开始前的状态:

```bash
$ git rebase --abort
```

## 工作区、暂存区、提交历史

在git里面有三个很重要的概念：工作区、暂存区、提交历史。

### 工作区（Working Directory）

就是你在电脑里能看到的目录，比如我的gitdemo文件夹就是一个工作区

### 暂存区（Stage）

一般存放在 ".git目录下" 下的index文件（.git/index）中，所以我们把暂存区有时也叫作索引（index）。
实际上指向暂存区的指针名就是index。

### 提交历史（Commit History）

隐藏目录`.git`其实是Git的版本库，也叫分支的提交历史。Git为我们自动创建的第一个分支master，
以及指向master的一个指针叫HEAD。请注意暂存区也是放在这个隐藏目录里面的。

![](https://xnstatic-1253397658.file.myqcloud.com/git01.jpg)

把文件往Git版本库里添加的时候，是分两步执行的：

1. 第一步是用`git add`把文件添加进去，实际上就是把文件修改添加到暂存区；
2. 第二步是用`git commit`提交更改，实际上就是把暂存区的所有内容提交到当前分支。

再次修改readme.txt:

```
hello git
I like git very much
add something
这次加一行啦啦啦
```

另外再添加一个文件LICENSE，内容如下:

```
MIT
```

先用`git status`查看一下状态:

```
[root@controller161 gitdemo]# git status
On branch master
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)

	modified:   readme.txt

Untracked files:
  (use "git add <file>..." to include in what will be committed)

	LICENSE

no changes added to commit (use "git add" and/or "git commit -a")
```

Git非常清楚地告诉我们，readme.txt被修改了，而LICENSE还从来没有被添加过，所以它的状态是Untracked

现在，使用两次命令`git add`或者`git add --all`，把readme.txt和LICENSE都添加后，用`git status`再查看一下：

```
[root@controller161 gitdemo]# git add --all
[root@controller161 gitdemo]# git status
On branch master
Changes to be committed:
  (use "git reset HEAD <file>..." to unstage)

	new file:   LICENSE
	modified:   readme.txt
```

现在，暂存区的状态就变成这样了:
![](https://xnstatic-1253397658.file.myqcloud.com/git02.jpg)

所以，`git add`命令实际上就是把要提交的所有修改放到暂存区（Stage），
然后，执行`git commit`就可以一次性把暂存区的所有修改提交到分支:

```
[root@controller161 gitdemo]# git commit -m "show stage work"
[master 32db843] show stage work
 2 files changed, 2 insertions(+)
 create mode 100644 contact.txt
```

一旦提交后，如果你又没有对工作区做任何修改，那么工作区就是"干净"的：

```
[root@controller161 gitdemo]# git status
On branch master
nothing to commit, working directory clean
```

现在版本库变成了这样，暂存区就没有任何内容了：
![](https://xnstatic-1253397658.file.myqcloud.com/git02.jpg)

## 关于diff

很多时候需要用diff命令来比较文件差异，总结一下:

* `git diff readme.txt`         -> 工作区 和 暂存区比较
* `git diff --cache readme.txt` -> 暂存区 和 版本库比较
* `git diff HEAD -- readme.txt` -> 版本库 和 工作区比较

如果`git diff`后面不加文件名readme.txt，表示要显示所有文件的差异。

## 撤销修改

有时候你也会犯傻修改了不该改的东西，这样时候可以通过`git checkout`命令撤销修改。

命令`git checkout -- readme.txt`意思就是，把readme.txt文件在工作区的修改全部撤销，这里有两种情况：

1. readme.txt自从修改后还没有被放到暂存区，现在，撤销修改就回到和版本库一模一样的状态；
2. readme.txt已经添加到暂存区后，又作了修改，现在，撤销修改就回到添加到暂存区时的状态。

总之，就是让这个文件回到最近一次git commit或git add时的状态。

`git checkout -- file` 命令中的`--`很重要，没有`--`，就变成了"切换到另一个分支"的命令，
我们在后面的分支管理中会再次遇到`git checkout`命令。

还有一种情况是，你想将暂存区的修改撤销掉:

```
git reset HEAD readme.txt
```

`git reset`命令既可以回退版本，也可以把暂存区的修改撤销掉。当我们用HEAD时，表示最新的版本。

注意：这里`git reset`并没有不会对工作区产生任何影响，只是撤销了暂存区的修改。

还记得如何丢弃工作区的修改吗？

```
git checkout -- readme.txt
```

整个世界终于清静了

总结一下撤销修改场景:

1. 场景1：当你改乱了工作区某个文件的内容，想直接丢弃工作区的修改时，用命令`git checkout -- file`
2. 场景2：当你不但改乱了工作区某个文件的内容，还添加到了暂存区时，想丢弃修改，分两步，
   第一步用命令`git reset HEAD file`，就回到了场景1，第二步按场景1操作。
3. 场景3：已经提交了不合适的修改到版本库时，想要撤销本次提交，参考版本回退一节，不过前提是没有推送到远程库。
4. 场景4：删除版本库里面的文件，但是工作区间内容保留，一般是先提交了想忽略的文件。`git reset --mixed 版本库ID`
5. 对于已提交至远程服务器的错误提交，参考下面的命令：

```
git reset --soft/mixed/hard <commit_id>
git push origin HEAD --force

HEAD   最新提交
HEAD~  上1个提交
HEAD~2 上2个提交
```

重要的事说三遍，最后有必要再次总结几个重要的指令：

* 当执行 `git reset HEAD` 命令时，暂存区的目录树会被重写，被 master 分支指向的目录树所替换，但是工作区不受影响。
* 当执行 `git rm --cached <file>` 命令时，会直接从暂存区删除文件，工作区则不做出改变。
* 当执行 `git rm -f <file>` 命令时，会直接把暂存区和工作区全部删了。
* 当执行 `git checkout .` 或者 `git checkout -- <file>` 命令时，会用暂存区全部或指定的文件替换工作区的文件。
  这个操作很危险，会清除工作区中未添加到暂存区的改动。
* 当执行 `git checkout HEAD .` 或者 `git checkout HEAD <file>` 命令时，
  会用 HEAD 指向的 master 分支中的全部或者部分文件替换暂存区和工作区中的文件。
  这个命令也是极具危险性的，因为不但会清除工作区中未提交的改动，也会清除暂存区中未提交的改动。

`git reset` 有三个选项，`--hard、--mixed、--soft`。

注意这三个选项对文件层面的`git reset`毫无作用，因为缓存区中的文件一定会变化，而工作目录中的文件一定不变。

 ```
 //仅仅只是撤销已提交的版本库，不会修改暂存区和工作区
 git reset --soft 版本库ID
 ``
 //仅仅只是撤销已提交的版本库和暂存区，不会修改工作区
 git reset --mixed 版本库ID
 
 //彻底将工作区、暂存区和版本库记录恢复到指定的版本库
 git reset --hard 版本库ID
 ```

注意这个版本库ID应该不是你刚刚提交的版本库ID，而是刚刚提交版本库的上一个版本库。

