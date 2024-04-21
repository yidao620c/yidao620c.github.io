---
title: git简明教程 - 技巧篇
date: 2017-02-15 09:34:21 +0800
toc: true
categories: [ 开发工具 ]
tags: [ git ]
---

这一篇会介绍git的一些常用技巧，开发中经常会遇到的问题，让我们感受git的强大之处。

**cherry-pick**我直接把它翻译成'摘樱桃'可以不？

`git cherry-pick`可以选择某一个分支中的一个或几个commit(s)来进行操作。假设我们有个稳定版本的分支master，
另外还有个开发版本的分支dev，我们不能直接把两个分支合并，这样会导致稳定版本混乱，但是又想增加一个dev中的功能到master中，
这里就可以使用cherry-pick了，其实也就是对已经存在的commit 进行再次提交。

简单用法：

```
git cherry-pick <commit id>
```

注意：当执行完 cherry-pick 以后，将会生成一个新的提交；这个新的提交的哈希值和原来的不同，但描述一样；

做一个简单演示，一个readme.txt文件、两个分支（master/dev）
<!-- more -->

master的提交历史如下：

```
$ git log --pretty=oneline
4bf41302efc10dd4a6e85c35ec8ef02b1b32e633 (HEAD -> master) second
1e2196ba347c4338bfa7589aabe2513ecabeeedd first
```

readme内容：

```
11111111111
22222222222222
```

dev的提交历史如下：

```
$ git log --pretty=oneline
16ae2df968afd92b89f97171480d764ec607deaf (HEAD -> dev) dev 5566
a25a08a678d55f7fa60c4185c2db38258aa9a700 dev 44444
9dc6d54e0357bfde4ccdc0b81888079d4fd4f585 dev 333
4bf41302efc10dd4a6e85c35ec8ef02b1b32e633 (master) second
1e2196ba347c4338bfa7589aabe2513ecabeeedd first
```

readme内容：

```
11111111111
22222222222222
dev 3333333333333
dev 44444444444444
dev 55555555555555555
dev 6666666666666666666
```

下面，我要把dev分支的16ae2df9合并到master中。

```
git checkout master
git cherry-pick 16ae2df9
```

如果出现冲突：

```
$ git cherry-pick a25a08...16ae2df9
error: a cherry-pick or revert is already in progress
hint: try "git cherry-pick (--continue | --quit | --abort)"
fatal: cherry-pick failed
```

先使用`git cherry-pick --abort`停掉之前的，然后再执行cherry-pick命令。

如果出现冲突：

```
$ git cherry-pick 16ae2df9
error: could not apply 16ae2df... dev 5566
hint: after resolving the conflicts, mark the corrected paths
hint: with 'git add <paths>' or 'git rm <paths>'
hint: and commit the result with 'git commit'
```

解决办法是先使用status命令看看什么文件冲突了：

```
$ git status
On branch master
You are currently cherry-picking commit a25a08a.
  (fix conflicts and run "git cherry-pick --continue")
  (use "git cherry-pick --abort" to cancel the cherry-pick operation)

Unmerged paths:
  (use "git add <file>..." to mark resolution)

        both modified:   readme.txt

no changes added to commit (use "git add" and/or "git commit -a")
```

打开readme文件看看：

```
11111111111
22222222222222
<<<<<<< HEAD
=======
dev 3333333333333
dev 44444444444444
dev 55555555555555555
dev 6666666666666666666
>>>>>>> 16ae2df... dev 5566
```

这里一开始我还有点疑惑，为什么会产生冲突呢，后来在知乎上找到了答案。

<https://www.zhihu.com/question/27804571>

cherry-pick只会将一个commit引入的更改应用到当前的commit上。而命令只会将从d1->d2的修改引入到m1中，这样当然会造成冲突。

如果只是想让master恢复到某个commit上，又不想有冲突，可使用：

```
$ git merge 16ae2df9 --squash
Updating 4bf4130..16ae2df
Fast-forward
Squash commit -- not updating HEAD
 readme.txt | 4 ++++
 1 file changed, 4 insertions(+)

$ git commit -m "merge"
[master a3a8255] merge
 1 file changed, 4 insertions(+)

$ cat readme.txt
11111111111
22222222222222
dev 3333333333333
dev 44444444444444
dev 55555555555555555
dev 6666666666666666666
```

接下来我再试试区间

```
# 下面是左开右闭区间
git cherry_pick <start-commit-id>...<end-commit-id>
# 下面是左右闭区间
git cherry-pick <start-commit-id>^...<end-commit-id>
```

基本一样的逻辑。




