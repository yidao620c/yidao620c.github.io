---
title: git简明教程 - 撤销篇
date: 2017-02-08 23:58:12 +0800
toc: true
categories: [ 开发工具 ]
tags: [ git ]
---

在实际使用git的过程中，我发现最常遇到的就是撤销，git里面有reset、checkout、revert来帮助我们撤回修改。
但是这几个命令有时候不是很好理解，虽然我在第一篇里面已经讲解过撤销，但是我还是想用专门用一篇来详细讲解如何撤销版本和文件。

之前只是讲了几个简单的命令教你怎么撤销，但是其中的原理如果你不懂的话就不会很好的使用这几个命令。
参考《Pro Git》第2版本中的重置揭密部分，为大家揭开撤销的原理和神秘面纱。
<!-- more -->

## 三棵树

理解 reset 和 checkout 的最简方法，就是以 Git 的思维框架（将其作为内容管理器）来管理三棵不同的树

 树                 | 用途                 
-------------------|--------------------
 HEAD              | 上一次提交的快照，下一次提交的父结点 
 Index             | 预期的下一次提交的快照        
 Working Directory | 沙盒                 

Git 主要的目的是通过操纵这三棵树来以更加连续的状态记录项目的快照，有时候也称为`工作目录`，`缓存区`和`提交历史`。
不管什么叫法其实意思都一样，你心里得记住这三个东东。

![](https://xnstatic-1253397658.file.myqcloud.com/git60.png)

`reset`、`checkout` 和 `revert`都可以作用于commit层面，并且`reset`、`checkout`还能作用于文件层面。

## commit层面

你传给 `git reset` 和 `git checkout` 的参数决定了它们的作用域。如果你没有包含文件路径，这些操作对所有提交生效。

### Reset

在提交层面上，reset 将一个分支的末端指向另一个提交。这可以用来移除当前分支的一些提交。
比如，下面这两条命令让 hotfix 分支向后回退了两个提交。

```
git checkout hotfix
git reset HEAD~2
```

hotfix 分支末端的两个提交现在变成了悬挂提交。也就是说，下次 Git 执行垃圾回收的时候，这两个提交会被删除。
换句话说，如果你想扔掉这两个提交，你可以这么做。reset 操作如下图所示：

reset之前：

![](https://xnstatic-1253397658.file.myqcloud.com/git61.png)

reset之后：

![](https://xnstatic-1253397658.file.myqcloud.com/git62.png)

如果你的更改还没有共享给别人，git reset 是撤销这些更改的简单方法。
当你开发一个功能的时候发现「糟糕，我做了什么？我应该重新来过！」时，reset 就像是 go-to 命令一样。

除了在当前分支上操作，你还可以通过传入这些标记来修改你的缓存区或工作目录：

* --soft – 缓存区和工作目录都不会被改变
* --mixed – 默认选项。缓存区和你指定的提交同步，但工作目录不受影响
* --hard – 缓存区和工作目录都同步到你指定的提交

当你传入 HEAD 以外的其他提交的时候要格外小心，因为 reset 操作会重写当前分支的历史。
正如 rebase 黄金法则所说的，在公共分支上这样做可能会引起严重的后果。

### Checkout

你应该已经非常熟悉提交层面的 `git checkout`。当传入分支名时，可以切换到那个分支。

```
git checkout hotfix
```

上面这个命令做的不过是将HEAD移到一个新的分支，然后更新工作目录和缓冲区。因为这可能会覆盖本地的修改，
Git 强制你提交或者暂存工作目录中的所有更改，不然在 checkout 的时候这些更改都会丢失，所以这个命令是很安全的。
和 `git reset` 不一样的是，`git checkout` 没有移动这些分支。

除了分支之外，你还可以传入提交的引用来 checkout 到任意的提交。这和 checkout 到另一个分支是完全一样的：把 HEAD 移动到特定的提交。
比如，下面这个命令会 checkout 到当前提交的祖父提交。

```
git checkout HEAD~2
```

![](https://xnstatic-1253397658.file.myqcloud.com/git63.png)

这对于快速查看项目旧版本来说非常有用。但如果你当前的 HEAD 没有任何分支引用，那么这会造成 HEAD 分离。这是非常危险的，如果你接着添加新的提交，
然后切换到别的分支之后就没办法回到之前添加的这些提交。因此，在为分离的 HEAD 添加新的提交的时候你应该创建一个新的分支。

在IDEA里面，有个`Checkout Revision`实际上就是调用的命令`git checkout revision`，回退到某个版本，但是如果你在上面进行修改了提交会报错：

![](https://xnstatic-1253397658.file.myqcloud.com/git64.png)

提示的很清楚，就是这是一个游离状态的HEAD指针提交，如果你非要这么做很可能后面会丢掉这个更改，最好能创建一个分支后再提交。

### Revert

Revert 撤销一个提交的同时会创建一个新的提交。这是一个安全的方法，因为它不会重写提交历史。比如，下面的命令会找出倒数第二个提交，
然后创建一个新的提交来撤销这些更改，然后把这个提交加入项目中。

```
git checkout hotfix
git revert HEAD~2
```

如下图所示

revert之前：

![](https://xnstatic-1253397658.file.myqcloud.com/git65.png)

revert之后：

![](https://xnstatic-1253397658.file.myqcloud.com/git66.png)

相比 `git reset`，它不会改变现在的提交历史。因此，`git revert` 可以用在公共分支上，`git reset` 应该用在私有分支上。

就像 `git checkout` 一样，`git revert` 也有可能会重写文件。所以，Git 会在你执行 revert 之前要求你提交或者暂存你工作目录中的更改。

## 文件层面

`git reset` 和 `git checkout` 命令也接受文件路径作为参数。这时它的行为就大为不同了。它不会作用于整份提交，参数将它限制于特定文件

### Reset

当检测到文件路径时，`git reset` 将缓存区同步到你指定的那个提交。下面这个命令会将倒数第二个提交中的 foo.py
加入到缓存区中，供下一个提交使用。

```
git reset HEAD~2 foo.py
```

和提交层面的 `git reset` 一样，通常我们使用HEAD而不是某个特定的提交。
运行 `git reset HEAD foo.py` 会将当前对 foo.py 文件的修改部分从缓存区中移除出去，而不会影响工作目录中对 foo.py 的更改。

![](https://xnstatic-1253397658.file.myqcloud.com/git67.png)

注意：`--soft`、`--mixed` 和 `--hard` 对文件层面的 `git reset` 毫无作用，因为缓存区中的文件一定会变化，而工作目录中的文件一定不变。

### Checkout

Checkout 一个文件和带文件路径 `git reset` 非常像，它除了会更新缓存区外，同时它还会更新工作区文件。
不像提交层面的 checkout 命令，它不会移动 HEAD引用，也就是你不会切换到别的分支上去。

![](https://xnstatic-1253397658.file.myqcloud.com/git68.png)

比如，下面这个命令将工作目录和缓存区中的 foo.py 同步到了倒数第二个提交中的 foo.py

```
git checkout HEAD~2 foo.py
```

和提交层面相同的是，它可以用来检查项目的旧版本，但作用域被限制到了特定文件。

这个命令通常和 HEAD 一起使用。比如 `git checkout HEAD foo.py` 等同于舍弃 foo.py 没有提交的更改。

## 总结

希望你现在熟悉并理解了 `reset`、`checkout` 和 `revert` 命令，你可能还是会有点困惑，毕竟不太可能记住不同调用的所有规则。

下面的速查表列出了命令对树的影响。 `HEAD` 一列中的 `REF` 表示该命令移动了 HEAD 指向的分支引用，而`HEAD` 则表示只移动了
HEAD 自身。
特别注意 `Safe?` 一列 - 如果它标记为 NO，那么运行该命令之前请考虑一下。

 撤销                       | HEAD | Index | WorkDir | Safe 
--------------------------|------|-------|---------|------
 **Commit Level**         |      |       |         |
 reset --soft [commit]    | REF  | NO    | NO      | YES  
 reset [commit]           | REF  | YES   | NO      | YES  
 reset --hard [commit]    | REF  | YES   | YES     | NO   
 checkout [commit]        | HEAD | YES   | YES     | YES  
 **File Level**           |      |       |         |
 reset (commit) [file]    | NO   | YES   | NO      | YES  
 checkout (commit) [file] | NO   | YES   | YES     | NO   


