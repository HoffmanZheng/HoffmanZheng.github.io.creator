---
title: "Git 与 Github 使用指北"
author: "Chenghao Zheng"
tags: ["Tools"]
categories: ["Study notes"]
date: 2019-12-26T21:49:47+01:00
draft: false
---

本篇介绍实用的工具 Git 及 Github 的常用功能及其操作，如：代码的版本控制，远程备份，团队项目协作等。

# Git 本地仓库

Git 工作流程：

![git工作流程](/images/work_process_git.png)

### 0. 应用场景

* 代码版本管理 / 多版本代码开发 / 多版本代码合并

### 1. 安装 Git

[参考伯克利课程 CS61b 中的教程安装 Git](https://sp18.datastructur.es/materials/lab/lab1setup/windows.html)

### 2. 配置 Git

```powershell
git config --global user.name + 你的英文名
git config --global user.email + 你的邮箱
git config --global push.default simple
git config --global core.quotepath false   // 不会对0×80以上的字符进行quote，中文显示正常。
git config --global core.editor "vim"
git config --global core.autocrlf input
git config --global --list                 // 查看当前配置列表
vi ~/.gitconfig                            // 编辑配置文件
```

### 3. Git 本地仓库

```shell
cd + 目标路径
git init                              // 初始化，当前目录创建 .git 文件夹
git add + 文件 / 文件夹 / .            // 选择要提交的内容
git commit -m "版本信息 / 更新日志"     // 写更新日志，提交代码快照
git commit                            // 进入默认编辑器vim更新日志
git log 
git reflog                            // 查看所有备份记录
git status
git reset --hard XXXXXX               // 版本回溯 写commit号前6位
git reset --hard HEAD^                // 回到上一个版本，可以多加^回到之前的版本。
```

注：
* 可以在目录下新建个 `.gitignore` 文件，把不需要 Ｇit 管理的文件/文件夹路径写进去。常见的有`.idea` `.vscode` `out` `*.iml` `target/` `build/` `.gradle/`  等。
* 在使用 `git reset` 回溯版本之前，一定要确保当前代码已经 `commit` 过了，因为这个操作会使当前没有 commit 的变动消失。

### 4. Git 分支管理

```shell
git branch                               // 查看现在存在的分支
git branch + newBranchName               // 新建分支，把当前代码快照复制到该分支
git checkout branchName / master         // 切换分支
history                                  // 查看所有操作记录
```

### 5. Git 合并分支

```shell
git checkout branchName / master        // 进入要保留的分支
git merge brachName                     // 将brach合并至当前分支
git add --> git commit
git branch -d branchName                // 删除分支
```

注：
* 合并分支时会检查代码是否有冲突，有冲突的话需要解决冲突，搜索 `====` 选择要保留的代码，再次合并。

### 6. 其他操作

```shell
start .                               // 在资源管理器打开当前文件夹
subl (文件 / 文件夹)                   // 快速打开sublime
git rm -r --cached + filename         // 删除add的路径
git add fileName --> git stash        // 暂时不像提交代码（通灵术）
git stash pop
```

# Github 远程仓库

### 0. 应用场景

* 代码云端备份 / 代码多地共享 / 团队协作 

### 1. SSH key 验证身份（关联Github账号）

```shell
ssh-keygen -t rsa -b 4096 -C + 你的邮箱           // 生成公钥文件
cat ~/.ssh/id_rsa.pub                          // 打开生成的公钥文件，复制公钥到网页
ssh -T git@github.com                          // 测试配对成功
```

注：

* 此操作旨在将本地 Git 与 Github 账号建立连接，在本地创建 `ssh key` 来创建连接，生成公钥、私钥，复制粘贴公钥到 Github上，`$cat id_rsa.pub` 显示出公钥复制粘贴到 Github 上，`头像-> setting-> SSH and GPG keys-> new ssh key`，这样就建立好了远程仓库和本地仓库的连接。
* 存在远程仓库，如何与本地建立连接：克隆该仓库的 ssh 地址。
* 存在本地项目，如何与远程仓库建立连接：初始化本地仓库 ---> `git remote add + 远程仓库ssh地址`

### 2. 远程仓库操作

```shell
git remote add origin git@xxxxxxx（ssh）       
git push -u origin master                      // 推送本地master分支到远程master分支
git pull                                       // 从远程同步到本地
// 上传其他分支
git push origin branchName:branchName
git checkout branchName --> git push -u origin branchName
git clone git@xxxxxx                           // 下载别人的代码，只能用 https
git remote -v                                  // 查看链接的远程仓库
git remote remove brachName                    // 删除仓库
git remote set-url <name> https：//xx          // 重置，如果之前弄错了 origin
```

想要删除远程仓库文件夹，可以这么操作：

```shell
git rm -r --cached + <filename>
git commit -m "delete remote file xx"
git push -u origin master      // 然后把该文件路径加到 .gitignore 中去
```

注：

* origin为远程仓库的名字，可以更改但不推荐。
* [clone下载速度很慢怎么办？](https://jscode.me/t/topic/789/2)
* Github国内的替代品有：
	* [腾讯云开发者平台](https://dev.tencent.com/)
	* [码云Gitee](https://gitee.com/)
	* [Gitlab](https://about.gitlab.com/)

### 3. Github 团队协作

* 在拥有对方仓库 读/写 权限的情况下

将对方仓库 `clone` 到本地进行修改，在对方仓库在新建 branch 进行 commit、push 操作，随后提交 `pull request` ，备注修改内容及意见。

* 在没有拥有对方仓库 读/写 权限的情况下

将对方仓库 `fork` 到自己仓库，再 clone 到本地进行修改，commit、push 之后到对方仓库提交 `pull request` ，备注修改内容及意见。

* 项目提交的原则，规范化的项目提交流程：
  * 使用 Github + 主干 / 分支模型进行开发，禁止直接 push 到 master 分支造成主分支损坏。
  * 一次提交不宜提交过多的内容，方便集中精力和 Code Review，同时也便于测试。

  * 遵循一切工作自动化原则：包括数据库建表、创造测试数据，几乎没有本地依赖，方便在其他人的机器上重现过程。

  * 使用自动化代码质量测试：checkstyle + spotbugs
  * `commit message` 应在第一行指出主要的修改内容，以动词开头不超过50字符，之后分条列举具体的修改过程，每行不超过72字符。

#  实用 Git 技巧

### 1. 快捷键设置

``` shell
vi ~/.bash_profile
alias ga="git add"
alias gc="git commit -v"
alias gp="git push"
alias gst="git status"
alias gco="git checkout"
alias glog="git log --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr)%Creset' --abbrev-commit --date=relative"                                    // 美化git log输出
alias glo="git log --oneline"
source ~/.bash_profile                                    // 让它生效
```

`git log` 美化前：

![](/images/gitlog美化前.png)

美化后：

![](/images/log美化后.png)

### 2. 简化 commit 提交历史

 在使用 Git 备份的时候，常常会遇到本次的修改对上一次的改动并不大，如果按照正常的 commit 流程的话，就会造成 commit 提交历史过多，扰乱思路难以选择的情况。因此，对于改动不大的提交，可以使用 ——amend 命令

```shell
git commit --amend
```

这将对上一次的 commit 做出修改并合并，是用新的 commit 替换掉原来的 commit，commit ID 会发生变化，有助于简化 commit 提交历史。

注意：不要修改已经 push 的 commit，除非是在自己的分支。

### 3. 使用 rebase 优化 commit 提交历史

对于已经提交的，杂乱无章的 commit，可以使用 rebase 来进行整理及优化。优于rebase操作较为危险，推荐先将主分支内容复制到新的分支进行操作。

```shell
git rebase -i HEAD~11        // interactive，修改当前commit前11个提交
```

commit 提交历史优化前：

![](/images/rebase优化前.png)

进入 rebase 交互界面后，提交历史的顺序会倒置，最新的提交在最下面，个人常用的优化选项为 fixup 和 edit，将左侧 pick 更改为相应的优化选项，保存退出后就进入优化流程。

![](/images/rebase具体操作.png)

进入优化流程后，分条修改 commit 内容，之后 continue 继续，直到走完全部的 rebase 流程。

![](/images/rebase流程.png)

commit 提交历史优化后：

![](/images/rebase之后.png)

将优化后的 commit push 到 Github：

```shell
git push -f rebaseOptimizeCommit:master
```

![](/images/rebase后同步到github.png)

### 4. 三种不同的 merge pull request 操作

在提交的 pull request 通过测试后，会进入选择 Merge pull request 的界面选择具体的合并方式：

![三种不同的 merge pull request](/images/三种merge2.png)

* `Create a merge commit`：会保留该 pull request 中的所有 commit 合并至主干，并额外产生一个 merge commit，之后在 git 的提交历史中可以看到所有的 commit 历史记录。

* `Squash and merge`：有一些很小的提交，或者是纠正前面的错误的提交，对于这类提交对整个工程来说不需要单独显示出来一次提交，不然导致项目的提交历史过于复杂；所以基于这种原因，我们可以把 dev 上的所有提交都合并成一个提交；然后提交到主干。

  需要注意的是，squash merge 并不会替你产生提交，而是产生一个新的 commit，需要你重新输出 commit 日志，再手动执行 git commit 操作；这个 commit **可能会变更提交者作者信息**，可能提交者并不是有原来 dev 分支上的开发人员，这将对代码实际贡献者的追溯产生困难。

* `Rebase and merge`：可手动选择需要 合并 / 保留 的 commit，并保留提交者的作者信息。直接把提交的几个 commit 平移到主分支上来。


