---
title: "Git 进阶与 GitHub 协作详解"
author: "Chenghao Zheng"
tags: ["tools"]
categories: ["Study notes"]
date: 2020-12-28T13:19:47+01:00
draft: false
---

在 [Git 与 GitHub 使用指北](https://hoffmanzheng.github.io/2019/git/) 中笔者已经介绍了 Git 及 GitHub 的基本操作，但对其原理不明所以，本篇将在之前的基础上结合 [Git Reference Manual](https://git-scm.com/docs) 介绍一些 Git 底层原理和高阶操作，以及在 GitHub 协作的流程。

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

Git 是一个分布式版本控制系统，当我们从 GitHub 上 `clone` 一个仓库到本地后，相当于把整个远程仓库全部复制到了本地仓库，包括它的所有历史状态。clone **不一定**要发生在网络上，它还可以在局域网或者磁盘上：

![](/images/git-clone.png)

#### push / pull

git push 能够成功的前提是 **远程仓库的状态一定要是本地待 push 状态的父亲**，相互之间没有发生任何的分叉。

当我们在 git push 失败后，通常会按照提示使用 git pull，随后会弹出一个合并窗口，这是因为 `git pull = git fetch + git merge`，fetch 会将远程仓库中所有的状态都拿到本地，merge 会从分叉点开始 **replay 重放** 到 origin 最新的提交，再执行合并过程。合并完成后，远程分支才成为本地分支的父亲，才可以继续完成 push 操作。

![](/images/git-merge-origin.png)

### 合并与冲突解决

#### Merge

Merge 会从提交状态的分叉点开始 **replay 重放** 到 `<commit>` 最新的提交，然后不同的仓库状态（代码快照/代码副本）进行合并，Git 会 **逐行检查冲突**，并尝试自动解决，当发现多个状态同时对某一行代码进行了修改时就会冲突报错，这时就需要通过人类的智慧挑选出正确的代码版本解决冲突，完成合并。

![](/images/git-merge-description.png)

Merge 的优点是简单，易于理解；记录了完整的历史；冲突解决简单（只需要解决一次冲突就可以）；缺点是历史（基于时间和提交记录的单线历史）杂乱无章；对 bisect 不友好。

#### rebase

Rebase 文档介绍说的是：基于另外一个分支的尖端，将当前分支的提交 **重演** 一遍（Reapply commits on top of another base tip）。之前讲 commit 的时候说过，commit 对象都是不可变的，rebase 的重演并不是将 A、B、C 移动到 G 上面去，而是 **产生了三个全新的提交** A' B' C'，所以这个时候提交历史被改变了。

![](/images/git-rebase.png)

每次重演的时候，需要依次解决遇到的所有的冲突，痛苦的是 ，同一个文件可能需要 **反复的去解决冲突** 。Rebase 的优点是分支历史是一条直线，清楚直观；对 bisect 友好；缺点是较为复杂，劝退新手；如果冲突可能要重复解决；会⼲扰别⼈（**和别⼈共享的分⽀永远不要force push**） 

![](/images/git-rebase-practice.png)

* 每次 rebase 之后都需要使用 force push  才能更新远程分支（因为分支分叉了），所以 **只能在自己的分支上 rebase 共用分支**（rebase master）。
* 在自己的分支 my-feature 上 rebase master 合并之后，也是无法 push 到 origin 的，因为和 origin/my-feature 分叉了，但是可以 force push，因为是自己的分支，不会影响到别人。
* 有个办法可以减少 rebase 时解决冲突的次数，使用 reset 恢复到 origin 分叉点，然后 commit 产生一个 “压扁的提交”（会丢失一些提交信息），再执行 rebase 的时候就只需要解决一次冲突就可以了。

因此 rebase 的使用场景有：定时将你的分⽀和主⼲进⾏同步。分支开发，时间长久，要合并时发现冲突太多了，因此需要定期同步。这个场景使用 merge 的话，提交的历史记录就会杂乱不堪， rebase 保持提交历史是一根直线的状态。

#### git merge --squash

将当前分支的所有提交压扁成一个再提交到共用分支上去，优点是把所有变更合在⼀起，更容易阅读；bisect友好；想要回滚或者 revert ⾮常⽅便；缺点是丢失了所有的历史记录。

![](/images/git-merge-squash.png)

然后在公共的分支上产生一个压扁后的提交记录：

![](/images/git-merge-squash-result.png)

### 回退与重放

#### checkout

Checkout 可以将代码 **复原到任何一个仓库状态** 上，且不修改仓库中的任何信息。在使用 tag 标签切换代码版本后，可能会提示版本 v5.0 不在任何一个分支上，即当前在 **脱离 HEAD** 的状态，所做的提交都会被遗弃，需要按照提示创建一个新的分支，才能保留创建的提交。

![](/images/git-checkout.png)

所以 checkout 最重要的作用就是查看代码的历史版本，此外还可以使用 `git checkout -- <file>` 来丢弃工作区中的修改。

![](/images/git-checkout--.png)

#### reset

Reset 会强行将当前分支 HEAD 指针移动到指定状态

* --soft：只移动指针，不碰 暂存区 / 工作区，使本地的修改得到了完全的保留
* --mixed（默认）：保留中间的文件变更，使其变成 not staged 状态（可以压扁提交，但会丢失中间的提交信息 ）
* --hard：清空当前的全部修改

#### cherry-pick

将某个提交在另外的分⽀上 **重放**，使用场景为将一个 bug fix 同步到较老的产品中或者将在 master 上进行的变更希望进入 release 环节。

先使用 checkout 切换到需要重放的分支上，然后 `git cherry-pick <commmitId>` 重放某个提交的修改，如果需要 cherry-pick 多个提交可以使用 `<commitId>..<commitId>` 来代表一个提交的区间。

#### revert 

产生一个 **反向提交**，撤销某个提交。使用场景为撤销历史中的某个更改，如 BUG 或不恰当的功能；或是回滚某次发布。`git revert <commitId>` ，revert 一个 revert 可以恢复之前的那次 revert。

### bisect 与 stash

#### stash

临时性将⼯作⽬录的变更储存起来，然后清空⼯作⽬录（类⽐「存档/读档」）。`git stash` （需要先使用 git add 添加到暂存区），`git stash pop`（把原先的更改从储藏室中恢复），`git stash list` 可以查看当前储藏的变更列表。

使用场景：

* 你正在开⼼的开发，突然来了⼀个线上 bug
* 你只好将⼿头的⼯作全部储存起来之后，开始别的⼯作
* 结果⼜来了⼀件优先级更⾼的 bug（再次 git stash，`git stash apply stash@{N}` 进行恢复）
* 完成别的⼯作之后回来继续

#### bisect

Bisect 用于在提交历史中查找某处引入的 bug，在问题可以稳定重现的时候，可以迅速定位出问题的代码，有的时候这比直接去 debug 更快捷。

1.  `git bisect start/reset`  
2. 切换提交历史后， `git bisect good/bad/skip`  
3. `git bisect run ./run.sh`  使用脚本实现自动化 bisect

#### bisect 学习案例

使用 git bisect 在 [Maven](https://github.com/apache/maven) 找出在 3.6.0 和 3.6.1 版本之间导致 [MNG-6700](https://issues.apache.org/jira/browse/MNG-6700) 的提交记录，可以使用 [MNG-6700 BUG Reproduction](https://github.com/hcsp/maven-issue-reproduction) 复现这个问题。

~~~shell
/** clone maven 项目到本地，切换版本，然后打包，解压包后就可以找到 mvn */
git clone https://github.com/apache/maven.git
git checkout maven-3.6.0
mvn clean package -Dmaven.test.skip=true -Drat.skip=true
unzip apache-maven/target/apache-maven-3.6.0-bin.zip -d .

/** 执行另外一个目录下的 maven 构建，使用 -f 参数，发现 3.6.0 版本没有出现问题 */
apache-maven-3.6.0/bin/mvn -f ../maven-issue-reproduction/pom.xml compile
[INFO] --- maven-compiler-plugin:3.1:compile (default-compile) @ maven-issue-reproduction ---
[INFO] Changes detected - recompiling the module!
[INFO] Compiling 1 source file to C:\Users\zch69\recipes\temp\maven\..\maven-issue-reproduction\target\classes
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  16.961 s


/** 使用 3.6.1 版本 compile 出错，复现问题 */
[INFO] ------------------------------------------------------------------------
[INFO] BUILD FAILURE
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  01:01 min
[INFO] Finished at: 2021-01-09T11:33:01+08:00
[INFO] ------------------------------------------------------------------------
[ERROR] Failed to execute goal org.jetbrains.kotlin:kotlin-maven-plugin:1.3.0:compile (compile) on project maven-issue-reproduction: Compilation failure: Compilation failure:
[ERROR] C:\Users\zch69\recipes\temp\maven-issue-reproduction\common\Os.kt:[3,12] Redeclaration: Os
[ERROR] C:\Users\zch69\recipes\temp\maven\..\maven-issue-reproduction\.\common\Os.kt:[3,12] Redeclaration: Os

/** 标记 3.6.1 为有问题的版本，开始 bisect */
git bisect bad
You need to start by "git bisect start"
Do you want me to do it for you [Y/n]? Y

/** 标记 3.6.0 为没有问题的版本后，bisect 将仓库状态自动切换到了提交历史 2928dc6b6 */
zch69@DESKTOP-BIRCN7U MINGW64 ~/recipes/temp/maven ((maven-3.6.1)|BISECTING)
$ git checkout maven-3.6.0
$ git bisect good
Bisecting: 27 revisions left to test after this (roughly 5 steps)
[2928dc6b68660cc5ac4022b0bfbc84d51d6905e4] refactoring: extracted initParent() method

zch69@DESKTOP-BIRCN7U MINGW64 ~/recipes/temp/maven ((2928dc6b6...)|BISECTING)
$ gst
HEAD detached at 2928dc6b6
You are currently bisecting, started from branch 'd66c9c0b3'.
  (use "git bisect reset" to get back to the original branch)
~~~

之后手动重复以上过程就可以找到导致这个问题的提交记录，但完全可以用脚本来 **自动化** 这个过程：

~~~shell
#!/bin/sh

VERSION=$(mvn help:evaluate -Dexpression=project.version -q -DforceStdout) 
rm -rf apache-maven-*
mvn clean package -Dmaven.test.skip=true -Drat.skip=true
unzip apache-maven/target/apache-maven-$VERSION-bin.zip -d .
apache-maven-$VERSION/bin/mvn -f ../maven-issue-reproduction/pom.xml compile
~~~

运行 `git bisect run ./run.sh` 后就可以等着最后的结果了：

~~~shell
8b7055fe3ff3696b821409a6904ff4d69aa3ff6b is the first bad commit
commit 8b7055fe3ff3696b821409a6904ff4d69aa3ff6b
Author: Mickael Istria <mistria@redhat.com>
Date:   Thu Nov 29 22:21:29 2018 +0100

    [MNG-6533] Prefer passing the interim project in ProjectBuildingResult

    Initialize the interim project with "simple" items (ie do not build
    not reference parent if it's not yet in the projectIndex) and returns
    it when installation fails further.
    This give a partial validation of the file, pretty convenient in IDEs.

 .../maven/project/DefaultProjectBuilder.java       | 46 ++++++++++++++++------
 1 file changed, 33 insertions(+), 13 deletions(-)
bisect run success
~~~

# Github 协作详解

GitHub 已经逐渐成为软件行业的一个事实标准，几乎所有的开源项目都可以在 GitHub 上找到代码，它已经不只是一个网站、一个平台了，可以说它已经是软件行业的图腾或者说是宗教了。

### GitHub 协作方式

GitHub 的出现，一改之前项目协作的困难，使得任何人发现了 BUG 都可以来帮助修改。但不能把主分支的 **写权限** 随便给外面的人，以免用户滥用把仓库搞乱。所以就出现了 `fork` 方式的协同合作：

![](/images/github-fork-process.png)

~~~shell
// fork 后的仓库并不会自动地随着上游仓库而更新，需要手动 sync a fork
git remote add upstream https://github.com/apache/maven.git   // 添加上游仓库
git fetch upstream  // 获取上游仓库的提交历史
git merge upstream/master   // 获取更新
~~~

### GitHub 社区

TODO