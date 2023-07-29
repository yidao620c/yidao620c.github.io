---
title: git简明教程 - 协作篇
date: 2017-02-08 22:52:36 +0800
toc: true
categories: [ 开发工具 ]
tags: [ git ]
---

Git是分布式版本控制系统，同一个Git仓库，可以分布到不同的机器上。怎么分布呢？最早，
肯定只有一台机器有一个原始版本库，此后，别的机器可以"克隆"这个原始版本库，
而且每台机器的版本库其实都是一样的，并没有主次之分。

实际情况往往是这样，找一台电脑充当服务器的角色，每天24小时开机，
其他每个人都从这个"服务器"仓库克隆一份到自己的电脑上，并且各自把各自的提交推送到服务器仓库里，
也从服务器仓库中拉取别人的提交。也就是说这个"服务器"只不过是用来作为一个桥梁，
提供给各个主机进行修改的交换。
<!-- more -->

## 使用GitHub

在GitHub出现以前，开源项目开源容易，但让广大人民群众参与进来比较困难，也只能把diff文件用邮件发过去，很不方便。

但是在GitHub上，利用Git极其强大的克隆和分支功能，广大人民群众真正可以第一次自由参与各种开源项目了。

* 在GitHub上，可以任意Fork开源仓库；
* 自己拥有Fork后的仓库的读写权限；
* 可以推送pull request给官方仓库来贡献代码。

关于GitHub的使用比较简单，参考[官方指南](https://guides.github.com/activities/hello-world/)，
大概20分钟就能看懂整个流程。

看完之后就可以自己创建一个远程仓库了，比如我现在创建了一个叫"gitdemo"的repository
![](https://xnstatic-1253397658.file.myqcloud.com/git13.png)
注意，我在初始化的时候还选择了创建一个README.md和一个LICENSE文件。

## 克隆远程仓库

先有远程仓库，直接通过远程仓库初始化本地仓库，比如:

```bash
git clone https://github.com/yidao620c/gitdemo.git
```

## 添加远程库

先有本地库，后有远程库的时候，如何关联远程库呢？就使用`git remote add`命令:

```bash
git remote add origin https://github.com/yidao620c/gitdemo.git
```

查看远程库信息:

```bash
[root@controller161 gitdemo]# git remote -v
origin	https://github.com/yidao620c/gitdemo.git (fetch)
origin	https://github.com/yidao620c/gitdemo.git (push)
```

上面显示了可以抓取和推送的origin的地址。如果没有推送权限，就看不到push的地址。

## 推送分支

推送分支，就是把该分支上的所有本地提交推送到远程库。推送时要指定本地分支，
Git就会把该分支推送到远程库对应的远程分支上:

```bash
git push origin master
```

如果推送的是dev分支就要这样写:

```bash
git push origin dev
```

但是，并不是一定要把本地分支往远程推送，那么，哪些分支需要推送，哪些不需要呢？

* master分支是主分支，因此要时刻与远程同步；
* dev分支是开发分支，团队所有成员都需要在上面工作，所以也需要与远程同步；
* bug分支只用于在本地修复bug，就没必要推到远程了，除非老板要看看你每周到底修复了几个bug；
* feature分支是否推到远程，取决于你是否和你的小伙伴合作在上面开发。

## 抓取分支

多人协作时，大家都会往master和dev分支上推送各自的修改。

这里我新创建的远程仓库还没有dev分支，这时候我先在本地创建dev分支，再push上去。
在dev分支上面，修改README.md内容如下:

```bash
# gitdemo
just for gitdemo test

dev branch add by "自己"
```

然后add，commit，push三部曲:

```bash
[root@controller161 gitdemo]# vim README.md
[root@controller161 gitdemo]# git add README.md
[root@controller161 gitdemo]# git commit -m "dev branch modify readme by self"
[dev 796e584] dev branch modify readme by self
 1 file changed, 2 insertions(+)
[root@controller161 gitdemo]# git push origin dev
...后面省略...
```

现在，模拟另外一个协作着，可以在另一个目录下克隆：

```bash
mkdir -p /root/work1 && cd /root/work1
git clone https://github.com/yidao620c/gitdemo.git
# 克隆单个分支
git clone -b mybranch --single-branch git://sub.domain.com/repo.git
```

现在我本地有两个仓库:

```bash
[root@controller161 gitdemo]# pwd
/root/work/gitdemo

[root@controller161 gitdemo]# pwd
/root/work1/gitdemo
```

不妨把`/root/work1/gitdemo`这个仓库使用者暂时称为"小明"，原来仓库使用者叫"自己"。

从远程库clone时，默认情况下小明只能看到本地的master分支，用git branch命令看看:

```bash
[root@controller161 gitdemo]# git branch
* master
```

现在，小明要在dev分支上开发，就必须创建远程origin的dev分支到本地，于是他用这个命令创建本地dev分支:

```bash
[root@controller161 gitdemo]# git checkout -b dev origin/dev
Branch dev set up to track remote branch dev from origin.
Switched to a new branch 'dev'
```

现在，小明就可以在dev上继续修改，然后，时不时地把dev分支push到远程。
他也修改了README.md文件，并且修改的同一行，内容如下:

```
# gitdemo
just for gitdemo test

dev branch add by "自己", add by xiao ming
```

然后也commit后push到dev上面:

```bash
[root@controller161 gitdemo]# vim README.md
[root@controller161 gitdemo]# git add README.md
[root@controller161 gitdemo]# git commit -m "modify readme by xiao ming"
[dev 10c9f30] modify readme by xiao ming
 1 file changed, 1 insertion(+), 1 deletion(-)
[root@controller161 gitdemo]# git push origin dev
...后面省略...
```

这时候你自己再次修改README.md，然后推送:

```
[root@controller161 gitdemo]# vim README.md
[root@controller161 gitdemo]# git add README.md
[root@controller161 gitdemo]# git commit -m "modify README again"
[dev 3c18f63] modify README again
 1 file changed, 1 insertion(+), 1 deletion(-)
[root@controller161 gitdemo]# git push origin dev
Username for 'https://github.com': yidao620c
Password for 'https://yidao620c@github.com':
To https://github.com/yidao620c/gitdemo.git
 ! [rejected]        dev -> dev (fetch first)
error: failed to push some refs to 'https://github.com/yidao620c/gitdemo.git'
hint: Updates were rejected because the remote contains work that you do
hint: not have locally. This is usually caused by another repository pushing
hint: to the same ref. You may want to first integrate the remote changes
hint: (e.g., 'git pull ...') before pushing again.
hint: See the 'Note about fast-forwards' in 'git push --help' for details.
```

提示很明显，你要先拉取最新代码合并才能push，好，那我先执行`git pull`合并:

```
[root@controller161 gitdemo]# git pull origin dev
remote: Counting objects: 3, done.
remote: Compressing objects: 100% (3/3), done.
remote: Total 3 (delta 0), reused 3 (delta 0), pack-reused 0
Unpacking objects: 100% (3/3), done.
From https://github.com/yidao620c/gitdemo
 * branch            dev        -> FETCH_HEAD
   796e584..10c9f30  dev        -> origin/dev
Auto-merging README.md
CONFLICT (content): Merge conflict in README.md
Automatic merge failed; fix conflicts and then commit the result.
```

自动合并失败，需要手动解决冲突，这时候我手动去解决README.md这个文件的冲突，方法和分支管理篇的解决方法一样。
先自己去修改这个文件，内容修改成如下:

```
# gitdemo
just for gitdemo test

1111111 dev branch add by "自己", add by xiao ming
```

改好了，再add, commit，push:

```
[root@controller161 gitdemo]# git add README.md
[root@controller161 gitdemo]# git commit -m "ok , I resolve confict"
[dev ad6fa66] ok , I resolve confict
[root@controller161 gitdemo]# git push origin dev
...后面省略...
```

因此，多人协作的工作模式通常是这样：

1. 首先，可以试图用git push origin branch-name推送自己的修改；
2. 如果推送失败，则因为远程分支比你的本地更新，需要先用git pull试图合并；
3. 如果合并有冲突，则解决冲突，并在本地提交；
4. 没有冲突或者解决掉冲突后，再用git push origin branch-name推送就能成功！
5. 如果git pull提示"no tracking information"，则说明本地分支和远程分支的链接关系没有创建，
   用命令`git branch --set-upstream branch-name origin/branch-name`

另外，如果执行命令`git pull` 报错信息为：

```
error: Your local changes to the following files would be overwritten by merge:
```

那么可以如下处理：

```
git stash  
git pull origin master  
git stash pop
```

然后再手动去解决冲突文件即可。

这就是多人协作的工作模式，一旦熟悉了，就非常简单。

再总结一下远程分支使用：

* 查看远程库信息，使用`git remote -v`；
* 本地新建的分支如果不推送到远程，对其他人就是不可见的；
* 从本地推送分支，使用`git push origin branch-name`，如果推送失败，先用git pull抓取远程的新提交；
* 在本地创建和远程分支对应的分支，使用`git checkout -b branch-name origin/branch-name`，本地和远程分支的名称最好一致；
* 建立本地分支和远程分支的关联，使用`git branch --set-upstream branch-name origin/branch-name`；
* 从远程抓取分支，使用`git pull`，如果有冲突，要先处理冲突。

## pull requests

这个是GitHub上面最主要的协作开发模式，有了这个普通平民也能参与开发顶级项目了。

如果你想对某个开源软件做贡献，就先fork这个仓库，这样就会在你自己的远程仓库中复制一份一模一样的仓库。

GitHub官网指南上面有个官方的仓库，专门用来给大家fork用，就是<https://github.com/octocat/Spoon-Knife>

我把它fork下来后，就是下面这样:
![](https://xnstatic-1253397658.file.myqcloud.com/git14.png)

然后你所有的修改就在自己的这个远程仓库上面完成，等你修改好以后。登陆GitHub，在项目主页，
点击"Pull requests" 菜单，点击"New pull request"按钮，填写你的修改说明提交就可以了。

至于你的这个Pull requests能不能被合并，就看该项目维护者的心情了。

## 保持fork之后的项目和上游同步

团队协作，为了规范，一般都是fork组织的仓库到自己帐号下，再提交pr，组织的仓库一直保持更新，
下面介绍如何保持自己fork之后的仓库与上游仓库同步。

下面以我 fork 团队的博客仓库为例

点击 fork 组织仓库到自己帐号下，然后就可以在自己的帐号下 clone 相应的仓库

使用 git remote -v 查看当前的远程仓库地址，输出如下：

```
origin  git@github.com:ibrother/staticblog.github.io.git (fetch)
origin  git@github.com:ibrother/staticblog.github.io.git (push)
```

可以看到从自己帐号 clone 下来的仓库，远程仓库地址是与自己的远程仓库绑定的（这不是废话吗）

接下来运行

```
git remote add upstream https://github.com/staticblog/staticblog.github.io.git
```

这条命令就算添加一个别名为 upstream（上游）的地址，指向之前 fork 的原仓库地址。git remote -v 输出如下：

```
origin   git@github.com:ibrother/staticblog.github.io.git (fetch)
origin   git@github.com:ibrother/staticblog.github.io.git (push)
upstream https://github.com/staticblog/staticblog.github.io.git (fetch)
upstream https://github.com/staticblog/staticblog.github.io.git (push)
```

之后运行下面几条命令，就可以保持本地仓库和上游仓库同步了

```
git fetch upstream
git checkout master
git merge upstream/master
```

或者更简单的命令：

```
git pull upstream {branch name}
```

接着就是熟悉的推送本地仓库到远程仓库

```
git push origin master
```

然后可以去github上自己的托管空间上创建pull request

## git工作流

这里讲最常见的三种工作流：

### Centralized Workflow

和svn类似，就一个master分支，push/pull循环
![](https://xnstatic-1253397658.file.myqcloud.com/git51.png)

### Feature Branch workflow

所有的feature开发都必须在特定的branch下而不是直接在master分之下开发

![](https://xnstatic-1253397658.file.myqcloud.com/git52.png)

feature分支开发完成后，发起merge requests，项目经理在code review后合并

Gitlab关于这种工作流的说明: <https://docs.gitlab.com/ee/workflow/workflow.html>

### Forking workflow

不同于使用唯一一个server-side repo作为中央库，这种工作流给每一个开发人员都定义分配一个server-side repo。
这意味着每一个contributor都有两个git repo:一个local one，一个public server-side one；

但是有个官方的official repo，大家的server-side repo都是从这个official repo通过fork操作获得。

![](https://xnstatic-1253397658.file.myqcloud.com/git53.png)

一般流程为：

* 项目经理初始化"official repo"
* 开发人员从official repo来做fork
* 开发人员从他们forked repo来做clone到本地
* 开发人员在他们自己的feature上进行开发工作
* git pull upstream master
* git push origin feature-branch
* 提交一个pull requests
* 项目经理合并开发人员申请的feature
* 开发人员和official repo保持同步更新
* 其他开发人员同步更新：git pull upstream master

### 三种工作流使用场景

1. Centralized Workflow => 习惯了svn，团队很小，项目简单
2. Feature Branch workflow => 一般的内部团队项目开发
3. Forking workflow => 大型分布式协作开发，开源软件

## 标签管理

最后还有标签就放这里讲吧，因为要涉及远程标签。
发布一个版本时，我们通常先在版本库中打一个标签（tag），这样就唯一确定了打标签时刻的版本，标签也是版本库的一个快照。
其实就是一个commit的别名而已，方便我们记忆，不如你说你要发布软件新版本号为"7b66ea9e"，这都什么跟什么啊。

创建标签:

```bash
git tag v1.0
```

查看所有标签:

```bash
git tag
```

默认标签是打在最新提交的commit上的。有时候，如果忘了打标签，比如现在已经是周五了，但应该在周一打的标签没有打，怎么办？
还记得以前的`git log`命令么，它能记录所有commit历史:

```
[root@controller161 gitdemo]# git log --pretty=oneline --abbrev-commit
ad6fa66 ok , I resolve confict
3c18f63 modify README again
10c9f30 modify readme by xiao ming
796e584 dev branch modify readme by self
dbf4ee5 Initial commit
```

比如你要对"modify readme by xiao ming"这个提交打个标签，就这样做:

```bash
git tag v0.2 10c9f30
```

查看单个标签:

```bash
git show v0.2
```

还可以创建带有说明的标签，用-a指定标签名，-m指定说明文字

```bash
git tag -a v0.3 -m "version 0.3 released" 3c18f63
```

再次使用`git show`就可以看到说明信息:

```
[root@controller161 gitdemo]# git show v0.3
tag v0.3
Tagger: xiongneng <xiongneng@winhong.com>
Date:   Mon Mar 6 17:00:06 2017 +0800

version 0.3 released

commit 3c18f63fb6d1ecf2e08608083050cd1a9d280667
Author: xiongneng <xiongneng@winhong.com>
Date:   Mon Mar 6 16:45:46 2017 +0800

    modify README again
```

标签还能用PGP签名，这里就不多讲了，好像暂时用不到。

删除标签:

```bash
git tag -d v0.2
```

推送标签到远程仓库:

```bash
git push origin v0.3
```

一次性推送全部尚未推送到远程的本地标签:

```bash
git push origin --tags
```

如果标签已经推送到远程，要删除远程标签就麻烦一点，先从本地删除：

```bash
git tag -d v0.3
Deleted tag 'v0.3' (was 6d3cf45)
```

然后，从远程删除，删除命令也是push，但是格式如下:

```bash
git push origin :refs/tags/v0.3
To https://github.com/yidao620c/gitdemo.git
 - [deleted]         v0.3
```

总结一下标签命令:

* 命令`git tag <name>`用于新建一个标签，默认为HEAD，也可以指定一个commit id；
* 命令`git tag -a <tagname> -m "blablabla..."`可以指定标签信息；
* 命令`git tag -s <tagname> -m "blablabla..."`可以用PGP签名标签；
* 命令`git tag`可以查看所有标签。
* 命令`git push origin <tagname>`可以推送一个本地标签；
* 命令`git push origin --tags`可以推送全部未推送过的本地标签；
* 命令`git tag -d <tagname>`可以删除一个本地标签；
* 命令`git push origin :refs/tags/<tagname>`可以删除一个远程标签。

## FAQ

怎样不合并拉取远程服务器最新的某个文件：

```
git fetch {remote}
git checkout FETCH_HEAD -- {file}
```

git 获取某个提交变更的文件，如果不加commit id表示最后一次提交：

```
# git show -r --pretty="" --name-status
M       README.md
A       image/dd.txt
D       test.json
```



