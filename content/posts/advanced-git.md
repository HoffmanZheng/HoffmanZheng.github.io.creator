---
title: "Git 进阶与 GitHub 协作详解"
author: "Chenghao Zheng"
tags: ["tools"]
categories: ["Study notes"]
date: 2020-12-28T13:19:47+01:00
draft: false
---

在 [Git 与 GitHub 使用指北](https://nervousorange.github.io/2019/git/) 中笔者已经介绍了 Git 及 GitHub 的基本操作，但对其原理不明所以，本篇将在之前的基础上结合 [Git Reference Manual](https://git-scm.com/docs) 介绍一些 Git 底层原理和高阶操作，以及 GitHub 的协作流程。

# Git 进阶

### 仓库、提交、分支的本质

#### Repository

Git 仓库的本质其实就是一个可以 **追溯所有历史** 的文件夹，历史的最小单位是仓库在某个时刻的状态（快照）。每一次对仓库的 commit 提交都将产生一个 `commitID`， 它代表着整个仓库所有的文件和文件夹在某个时刻的状态 / 副本 / 代码快照，所以仓库的本质即是所有这些快照的集合。

可以通过 `git log` 来查看仓库历史上的状态，然后使用 `git checkout <commitID>` 来使仓库恢复到某个状态。在 GitHub 网站上可以使用 `https://github.com/<userName>/<repositoryName>/tree/<commitID/tag/branch>` 看到这个仓库在某个时刻的状态。

以下几种方式都可以标识仓库的状态：

* 提交的 commitID，是一个十六进制字符串的 SHA-1
* 分支名 branch，指向分支最新一次的提交
* HEAD，指向当前分支的头
* 标签 tag，是针对某个仓库状态打的标签，`git tag v1.0` 

> 相关阅读：[GeeksforGeeks：What is a GIT Repository?](https://www.geeksforgeeks.org/what-is-a-git-repository/)

#### Commit

commit 提交是指将仓库从一个状态转移到另外一个状态时发生的变化，它产生的 commit 对象包括：

1. commitID
2. parent-id，提交的父 id
3. commitor 把代码提交到仓库的人（通常是对仓库有写权限的人），包括他的 name、email
4. author 编写代码的人（贡献者），包括他的 name、email

commit 对象是 **不可变** 的，`git commit --amend` 并没有修改原来的 commit，而是创建了一个新的提交，来替换当前节点（当前分支上最后一次 commit），并连到当前节点的上一个节点。如果 amend 已经 push 在 GitHub 上的提交，会导致和远程仓库的分支 **分叉**，导致无法向远程仓库 push 新的提交（解决方法见 push / pull）。

![](/images/git-amend.png)

#### Branch

分支是指向某个提交对象的，一个可以 **移动的指针**。新建一个分支，就是新建了一个指针。可以通过 `git checkout -b <new-branch> {仓库的某个状态}` 来基于仓库的任何一个状态创建一个新的分支。

`HEAD` 指向当前分支的头，HEAD~{N} 符号 tilda 表示 HEAD 的第 N 任祖先，当然也可以是 master~1 或者 `<commitId>~{N}` ，HEAD^{N} 符号 caret 表示从左往右数第 N 个父亲。持续的工作和提交，会在仓库中形成了一棵树。

![](/images/git-HEAD.png)

### 远程仓库、push 与 pull 详解

#### Github Logo 的由来

由于一个仓库的状态可以有多个父亲，也可以把多个提交合并在一起，称之为 octopus merge 章鱼合并，之后就演变成了 octopuss = octopus + pussy ---> `octocat` 八爪猫，成为今天 GitHub 的 Logo。

![](/images/git-octocat.png)

#### 远程仓库

Git 是一个分布式版本控制系统，当我们从 GitHub 上 `clone` 一个仓库到本地后，相当于把整个远程仓库全部复制到了本地仓库，包括它的所有历史状态。clone 不一定要发生在网络上，它还可以在局域网或者磁盘上：

![](/images/git-clone.png)

#### push / pull

git push 能够成功的前提是 **远程仓库的状态一定要是本地待 push 状态的父亲**，相互之间没有发生任何的分叉。

![](/images/git-pull.png)

当我们在 git push 失败后，通常会按照提示使用 git pull，随后会弹出一个合并窗口，这是因为 `git pull = git fetch + git merge`，fetch 会将远程仓库中所有的状态都拿到本地，merge 会从分叉点开始 **replay 重放** 到 origin 最新的提交，再执行合并过程。合并完成后，远程分支才成为本地分支的父亲，才可以继续完成 push 操作。

![](/images/git-merge-origin.png)

### 合并与冲突解决

#### Merge

Merge 会将多个仓库的状态（代码快照/代码副本）进行合并，Git 会逐行检查冲突，并尝试自动解决，当发现多个状态同时对某一行代码进行了修改时就会冲突报错，这时就需要通过人类的智慧挑选出正确的代码版本解决冲突，完成合并。



rebase squash

### 回退与重放

reset revert cherry-pick

### bisect 与 stash



# Github 协作详解

