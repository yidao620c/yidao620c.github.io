---
title: Git常用操作命令
date: 2017-02-25 13:50:23 +0800
toc: true
categories: [ 开发工具 ]
tags: [ git ]
---

这里记录下日常工作中git的一些常用操作命令，以及一些常见FAQ，作为参考手册。

<!-- more -->

## 分支操作

```bash
# 拉取所有远程分支信息：
git pull --all
# 创建新的本地分支并跟踪远程分支
git checkout --track origin/branch
# 打标签
git tag <tag> -m "tag说明"
# 推送标签到远程仓库
git push origin <tag>
# 删除本地分支
git push origin --delete branch
# 新增远程仓库地址
git remote add origin remote_url
# 更新远程仓库地址
git remote set-url origin new_remote_url
```

## 版本回退

版本号回退有两种情况，一种是`git reset`（回退，不保留提交记录），一种是`git revert`（反做，保留提交记录）。
先使用`git log`查看版本提交记录历史，确认要回退的提交ID号。

```bash
# 本地回退
git reset --hard 提交ID
# 远程回退
git push -f origin master
# 本地反做
git revert -n 提交ID
# 远程反做
git push origin master
```

## 暂存当前编辑内容

有时候本地正在编辑内容，但是又不想提交。这时候又要切换到其他分支去做事情。 这时候可以将本地修改暂存起来，
后面切换回来再恢复。

```bash
# 把所有未提交的修改保存起来
git stash
# 恢复之前暂存的内容
git stash pop
# 查看当前暂存堆栈内容列表
git stash list
# 删除暂存堆栈中某一个暂存记录
git stash drop stash@{0}
# 删除所有暂存记录
git stash clear
```

## 合并多次提交为一个

```bash
# 先通过命令查看历史记录
git log
# 然后合并到指定提交
git rebase -i 提交ID
# 合并最近3次提交
git rebase -i HEAD~3
```

## 创建内容为空分支

有时候可以从头创建一个新分支，里面啥东西都没有。

```bash
git checkout --orphan branch
git reset --hard
```

## Windows下Git账号管理

```bash
git config --global credential.helper manager
```

其中`~/.gitconfig`配置如下

```
[user]
    name = xiongneng
    email = xiongneng@xxx.com

[credential]
    helper = manager

[http]
    sslVerify = false
```

第一次的时候回提示用户输入口令，成功后会存储到Windows凭据管理器中，以后就不需要输入口令了。 
后面更新git口令后，需要在`控制面板\用户账号\凭据管理器`下面去修改即可。

## FAQ
记录下git操作过程中出现的问题和解决办法

### 本地push出现`RPC failed; curl 56 OpenSSL SSL_read: Connection was reset`
如图
![img.png](https://xnstatic-1253397658.file.myqcloud.com/20230715-18.png)

原因：push过程中更新的文件过大，需要设置本地push的缓存区大小。

解决步骤：

1、首先输入如下命令：
```bash
git config --globle http.sslVerify "false"
```

2、文件大小的上限设置：
```bash
git config --global http.postBuffer 524288000
```

3、如果还是push失败，则需要继续修改git缓存的大小。

