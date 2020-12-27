---
title: "Git 进阶与 GitHub 协作详解"
author: "Chenghao Zheng"
tags: ["tools"]
categories: ["Study notes"]
date: 2020-12-19T13:19:47+01:00
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

commit 对象是 **不可变** 的，`git commit --amend` 并没有修改原来的 commit，而是创建了一个新的提交，来替换当前节点（当前分支上最后一次 commit），并连到当前节点的上一个节点。如果 amend 已经 push 在 GitHub 上的提交，会导致和远程仓库的分支分叉，导致无法向远程仓库 push 新的提交。

![](/images/git-amend.png)

#### Branch

分支是指向某个提交对象的，一个可以移动的 **指针**。新建一个分支，就是新建了一个指针。可以通过 `git checkout -b <new-branch> {仓库的某个状态}` 来基于仓库的任何一个状态创建一个新的分支。  

持续的工作和提交，在仓库中建立了一棵树，

### 合并与冲突

rebase squash

### 回退与重放

reset revert cherry-pick

### bisect 与 stash



### 





# Github 协作详解

### Github Logo 的由来

### 本地仓库、远程仓库、push 与 pull 详解

