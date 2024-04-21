---
title: git简明教程 - 分支篇
date: 2017-02-07 22:08:25 +0800
toc: true
categories: [ 开发工具 ]
tags: [ git ]
---

终于要介绍git的杀手级特性分支了，这也是大部分人使用git的原因。
其他版本控制系统如SVN等都有分支管理，但是用过之后你会发现，这些版本控制系统创建和切换分支比蜗牛还慢，
简直让人无法忍受，结果分支功能成了摆设，大家都不去用。
Git的分支是与众不同的，无论创建、切换和删除分支，Git在1秒钟之内就能完成！无论你的版本库是1个文件还是1万个文件。
<!-- more -->

## 创建与合并分支

每次提交，Git都把它们串成一条时间线，这条时间线就是一个分支。截止到目前，只有一条时间线，
在Git里，这个分支叫主分支，即master分支。HEAD严格来说不是指向提交，
而是指向master，master才是指向提交的，所以，HEAD指向的就是当前分支。

一开始的时候，master分支是一条线，Git用master指向最新的提交，再用HEAD指向master，就能确定当前分支，以及当前分支的提交点：
![](https://xnstatic-1253397658.file.myqcloud.com/git04.png)

每次提交，master分支都会向前移动一步，这样，随着你不断提交，master分支的线也越来越长。

当我们创建新的分支，例如dev时，Git新建了一个指针叫dev，
指向master相同的提交，再把HEAD指向dev，就表示当前分支在dev上：
![](https://xnstatic-1253397658.file.myqcloud.com/git05.png)

Git创建一个分支很快，就是增加一个dev指针，改改HEAD的指向。

不过，从现在开始，对工作区的修改和提交就是针对dev分支了，比如新提交一次后，dev指针往前移动一步，而master指针不变
![](https://xnstatic-1253397658.file.myqcloud.com/git06.png)

假如我们在dev上的工作完成了，就可以把dev合并到master上。Git怎么合并呢？
最简单的方法，就是直接把master指向dev的当前提交，就完成了合并：
![](https://xnstatic-1253397658.file.myqcloud.com/git07.png)

所以Git合并分支也很快！就改改指针，工作区内容也不变。

合并完分支后，甚至可以删除dev分支。删除dev分支就是把dev指针给删掉，删掉后，我们就剩下了一条master分支:
![](https://xnstatic-1253397658.file.myqcloud.com/git08.png)

下面开始实战演练一下。

首先，我们创建dev分支，然后切换到dev分支:

```bash
[root@controller161 gitdemo]# git checkout -b dev
Switched to a new branch 'dev'
```

`git checkout`命令加上-b参数表示创建并切换，相当于以下两条命令:

```bash
git branch dev
git checkout dev
```

然后用`git branch`命令查看当前分支:

```bash
[root@controller161 gitdemo]# git branch
* dev
  master
```

`git branch`命令会列出所有分支，当前分支前面会标一个*号。

然后，我们就可以在dev分支上正常提交，比如对readme.txt做个修改，加上一行:

```bash
hello git
I like git very much
add something
这次加一行啦啦啦
this is dev line
```

然后提交：

```bash
[root@controller161 gitdemo]# git add readme.txt
[root@controller161 gitdemo]# git commit -m "dev commit 1"
[dev 97e84a1] dev commit 1
 1 file changed, 1 insertion(+)
```

现在，dev分支的工作完成，我们就可以切换回master分支:

```bash
[root@controller161 gitdemo]# git checkout master
Switched to branch 'master'
```

切换回master分支后，再查看一个readme.txt文件，刚才添加的内容不见了！
因为那个提交是在dev分支上，而master分支此刻的提交点并没有变:
![](https://xnstatic-1253397658.file.myqcloud.com/git08.png)

现在，我们把dev分支的工作成果合并到master分支上:

```bash
[root@controller161 gitdemo]# git merge dev
Updating 240afec..97e84a1
Fast-forward
 readme.txt | 1 +
 1 file changed, 1 insertion(+)
```

git merge命令用于合并指定分支到当前分支。合并后，再查看readme.txt的内容，就可以看到，和dev分支的最新提交是完全一样的

合并完成后，就可以放心地删除dev分支了:

```bash
[root@controller161 gitdemo]# git branch -d dev
Deleted branch dev (was 97e84a1)
```

删除后，查看branch，就只剩下master分支了:

```bash
[root@controller161 gitdemo]# git branch
* master
```

因为创建、合并和删除分支非常快，所以Git鼓励你使用分支完成某个任务，
合并后再删掉分支，这和直接在master分支上工作效果是一样的，但过程更安全。

## 解决冲突

冲突经常会发生，我们在不同分支上面修改了同一个文件，最后合并的时候肯定会有冲突。现在我们演示下冲突。

先开一个feature1分支:

```bash
[root@controller161 gitdemo]# git checkout -b feature1
Switched to a new branch 'feature1'
```

在feature1分支上面修改readme.txt如下:

```
hello git, feature1
I like git very much
add something
这次加一行啦啦啦
this is dev line
```

然后提交:

```bash
[root@controller161 gitdemo]# git add --all
[root@controller161 gitdemo]# git commit -m "feature1 111"
[feature1 f0e23f8] feature1 111
 1 file changed, 1 insertion(+), 1 deletion(-)
```

切换到master分支，修改同样的文件readme.txt,内容如下:

```
hello git master
I like git very much
add something
这次加一行啦啦啦
this is dev line
```

然后也提交:

```bash
[root@controller161 gitdemo]# git add readme.txt
[root@controller161 gitdemo]# git commit -m "master"
[master 18c3eb0] master
 1 file changed, 1 insertion(+), 1 deletion(-)
```

现在，master分支和feature1分支各自都分别有新的提交，变成了这样:
![](https://xnstatic-1253397658.file.myqcloud.com/git10.png)

这种情况下，Git无法执行"快速合并"，试试合并看看:

```bash
[root@controller161 gitdemo]# git merge feature1
Auto-merging readme.txt
CONFLICT (content): Merge conflict in readme.txt
Automatic merge failed; fix conflicts and then commit the result.
```

readme.txt文件存在冲突，必须手动解决冲突后再提交。`git status`也可以告诉我们冲突的文件:

```bash
[root@controller161 gitdemo]# git status
On branch master
You have unmerged paths.
  (fix conflicts and run "git commit")

Unmerged paths:
  (use "git add <file>..." to mark resolution)

	both modified:   readme.txt

no changes added to commit (use "git add" and/or "git commit -a")
```

打开readme.txt，内容变成这样:

```
<<<<<<< HEAD
hello git master
=======
hello git, feature1
>>>>>>> feature1
I like git very much
add something
这次加一行啦啦啦
this is dev line
```

Git用<<<<<<<，=======，>>>>>>>标记出不同分支的内容，这是典型的diff冲突格式，我们修改如下后保存:

```bash
hello git master and feature1
I like git very much
add something
这次加一行啦啦啦
this is dev line
```

再提交:

```bash
[root@controller161 gitdemo]# git add readme.txt
[root@controller161 gitdemo]# git commit -m 'fix confict'
[master a2c9e32] fix confict
```

现在，master分支和feature1分支变成了下图所示:
![](https://xnstatic-1253397658.file.myqcloud.com/git12.png)

查看合并图:

```bash
[root@controller161 gitdemo]# git log --graph --pretty=oneline --abbrev-commit
*   a2c9e32 fix confict
|\
| * f0e23f8 feature1 111
* | 18c3eb0 master
|/
* 97e84a1 dev commit 1
* 240afec show stage work
* 32db843 show stage work
* 1d57c05 modify readme
* 50f5fdc readme file
```

如果想保留某个分支的提交记录，删除后也有记录历史。可禁用Fast forward:

```bash
git merge --no-ff -m "merge with no-ff" dev
```

本次合并要创建一个新的commit，所以加上-m参数，把commit描述写进去。

## 分支策略

在实际开发中，我们应该按照几个基本原则进行分支管理:

首先，master分支应该是非常稳定的，也就是仅用来发布新版本，平时不能在上面干活；

那在哪干活呢？干活都在dev分支上，也就是说，dev分支是不稳定的，到某个时候，比如1.0版本发布时，
再把dev分支合并到master上，在master分支发布1.0版本；

你和你的小伙伴们每个人都在dev分支上干活，每个人都有自己的分支，时不时地往dev分支上合并就可以了，类似这样:
![](https://xnstatic-1253397658.file.myqcloud.com/git121.png)

## Bug分支

bug分支一般是那种时间短任务急的分支，修改完马上要合并提交的，就是临时分支。
但是你一般都在dev分支上面工作，这时候有些工作还没有提交。

```bash
[root@controller161 gitdemo]# vim readme.txt
[root@controller161 gitdemo]# git branch
* dev
  master
[root@controller161 gitdemo]# git status
On branch dev
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)

	modified:   readme.txt

no changes added to commit (use "git add" and/or "git commit -a")
```

并不是你不想提交，而是工作只进行到一半，还没法提交，预计完成还需1天时间。但是，必须在两个小时内修复该bug，怎么办？

幸好，Git还提供了一个stash功能，可以把当前工作现场"储藏"起来，等以后恢复现场后继续工作:

```bash
[root@controller161 gitdemo]# git stash
Saved working directory and index state WIP on dev: a2c9e32 fix confict
HEAD is now at a2c9e32 fix confict
```

可以放心地创建分支来修复bug，假定需要在master分支上修复，就从master创建临时分支:

```bash
[root@controller161 gitdemo]# git checkout master
Switched to branch 'master'
[root@controller161 gitdemo]# git checkout -b issue-101
Switched to a new branch 'issue-101'
```

现在修复bug，readme.txt修改为:

```
hello git master and feature1
I like git very much
add something, fix bug101
这次加一行啦啦啦
this is dev line
```

然后提交:

```bash
[root@controller161 gitdemo]# git add readme.txt
[root@controller161 gitdemo]# git commit -m "fix bug101"
[issue-101 a4401b2] fix bug101
 1 file changed, 1 insertion(+), 1 deletion(-)
```

修复完成后，切换到master分支，并完成合并，最后删除issue-101分支:

```bash
[root@controller161 gitdemo]# git merge --no-ff -m "merged bug fix 101" issue-101
Merge made by the 'recursive' strategy.
 readme.txt | 2 +-
 1 file changed, 1 insertion(+), 1 deletion(-)
```

改完bug，是时候接着回到dev分支干活了:

```bash
[root@controller161 gitdemo]# git checkout dev
Switched to branch 'dev'
[root@controller161 gitdemo]# git status
On branch dev
nothing to commit, working directory clean
```

切换回来后第一件事就是把刚刚的bug分支合并:

```bash
git merge --no-ff -m "dev-merge-m" master
```

工作区是干净的，刚才的工作现场存到哪去了？用git stash list命令看看:

```bash
[root@controller161 gitdemo]# git stash list
stash@{0}: WIP on dev: a2c9e32 fix confict
```

`git stash pop`，恢复的同时把stash内容也删了（如果提示冲突，去文件手动改正）:

```bash
[root@controller161 gitdemo]# git stash pop
On branch dev
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)

	modified:   readme.txt

no changes added to commit (use "git add" and/or "git commit -a")
Dropped refs/stash@{0} (2df611a386511cc7989db79457b620b9971c08b5)
```

你可以多次stash，恢复的时候，先用git stash list查看，然后恢复指定的stash，用命令:

```bash
git stash apply stash@{0}
```

有必要再次总结一下。

master合并merge解决好的bug后，不要先把dev解印，先合并master，获取里面的bug方案后，在解印。解印时会有提示冲突，需手动改一次文件。

1. 在dev下正常开发中，说有1个bug要解决，首先我需要把dev分支封存stash
2. 在master下新建一个issue-101分支，解决bug，成功后
3. 在master下合并issue-101
4. 在 dev 下合并master， 这样才同步了里面的bug解决方案
5. 解开dev封印stash pop，系统自动合并 & 提示有冲突，因为封存前dev写了东西，此时去文件里手动改冲突
6. 继续开发dev，最后add，commit
7. 在master下合并最后完成的dev

代码过程如下：

```bash
1. $ git stash
2. $ git checkout master
   $ git checkout -b issue-101
   //去文件里修bug
   $ git add README.md
   $ git commit -m "fix-issue-101"
3. $ git checkout master
   $ git merge --no-ff -m "m-merge-issue-101" issue-101
   $ git branch -d issue-101
4. $ git checkout dev
   $ git merge --no-ff -m "dev-merge-m" master
5. $ git stash pop
   //提示冲突，去文件手动改正
   Auto-merging README.md
   CONFLICT (content): Merge conflict in README.md
6. //继续开发 ... ... ，完成后一并提交
   $ git add README.md
   $ git commit -m "fixconflict & append something"
7. $ git checkout master
   $ git merge --no-ff -m "m-merge-dev" dev
   $ git branch -d dev
```

开发一个新feature，最好新建一个分支。

如果要丢弃一个没有被合并过的分支，可以通过git branch -D <name>强行删除。


