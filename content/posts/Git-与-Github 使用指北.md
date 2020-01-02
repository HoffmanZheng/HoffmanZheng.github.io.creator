---
title: "Git 与 Github 使用指北"
author: "Chenghao Zheng"
tags: ["Tools"]
categories: ["Study notes"]
date: 2019-12-26T21:49:47+01:00
draft: false
---

本篇介绍实用的工具 Git 及 Github 的常用功能及其操作，如：代码的版本控制，远程备份，团队项目协作等。



### 一、Git 本地仓库（版本管理）

* 应用场景一：

```shell
甲方：我这个背景颜色要五彩斑斓的黑
甲方：我还是想要土耳其沙滩上空蓝天的那种蓝色，你再改下
甲方：我还是觉得原先设计的那种颜色比较好看，改回去把
我：好的（MMP，不早说，我现在已经没法撤销了！）
```

* 应用场景二：

```shell
老板：给我做一个页面
我：花了几天，做出来了，请过目
老板：不够醒目，再改改
老板娘：我觉得挺好，要是背景能做成五彩斑斓的黑就好了
我：好嘞，我做两份你们对比一下（MMP，你们俩就不能统一意见吗！）
```

* 应用场景三：

```shell
老板：昨天夜里我和老板娘达成一致了，两个版本都要，我的放在上面，老板娘的在下面
我：好的，我合并一下 (￣▽￣)"
```

* Git 工作流程：

  ![git工作流程](/images/work_process_git.png)

#### 0. 安装 Git

[参考伯克利课程 CS61b 中的教程安装 Git](https://sp18.datastructur.es/materials/lab/lab1setup/windows.html)

#### 1. 配置 Git

```shell
git config --global user.name + 你的英文名
git config --global user.email + 你的邮箱
git config --global push.default simple
git config --global core.quotepath false   // 不会对0×80以上的字符进行quote，中文显示正常。
git config --global core.editor "code --wait"
git config --global core.autocrlf input
git config --global --list                 // 查看当前配置列表
vi ~/.gitconfig                            // 编辑配置文件
```

#### 2. Git 本地仓库（应用场景一）

```shell
cd + 目标路径
git init                              // 初始化，当前目录创建 .git 文件夹
git add + 文件 / 文件夹 / .            // 选择要提交的内容
git commit -m "版本信息 / 更新日志"     // 写更新日志，提交代码快照
git log 
git reflog                            // 查看所有备份记录
git status
git reset --hard XXXXXX               // 版本回溯 写commit号前6位
git reset --hard HEAD^                // 回到上一个版本，可以多加^回到之前的版本。
```

注：
* 可以在目录下新建个 `.gitignore` 文件，把不需要 `git` 管理的文件/文件夹路径写进去。常见的有`.idea` `.vscode` 等。
* 在使用 `git reset` 回溯版本之前，一定要确保当前代码已经 `commit` 过了，因为这个操作会使当前没有 `commit` 的变动消失。

#### 3. Git 分支管理（应用场景二）

```shell
git branch                               // 查看现在存在的分支
git branch + newBranchName               // 新建分支，把当前代码快照复制到该分支
git checkout branchName / master         // 切换分支
history                                  // 查看所有操作记录
```

#### 4. Git 合并分支（应用场景三）

```shell
git checkout branchName / master        // 进入要保留的分支
git merge brachName                     // 将brach合并至当前分支
git add --> git commit
git branch -d branchName                // 删除分支
```

注：
* 合并分支时会检查代码是否有冲突，有冲突的话需要解决冲突，搜索 `====` 选择要保留的代码，再次合并。

#### 5. 其他操作

```shell
start .                               // 在资源管理器打开当前文件夹
subl (文件 / 文件夹)                   // 快速打开sublime
git rm -r --cached + filename         // 删除add的路径
git add fileName --> git stash        // 暂时不像提交代码（通灵术）
git stash pop
```



### 二、Github 远程仓库（云端备份）

* 应用场景

```shell
我需要同时在公司和家里写代码
笔记本被女朋友的奶茶泡坏了
不小心运行了 rm -rf/
在云端实时备份代码就能解决以上所有问题呢 (ง •̀-•́)ง
```

#### 1. SSH key 验证身份（关联Github账号）

```shell
ssh-keygen -t rsa -b 4096 -C + 你的邮箱           // 生成公钥文件
cat ~/.ssh/id_rsa.pub                          // 打开生成的公钥文件，复制公钥到网页
ssh -T git@github.com                          // 测试配对成功
```

注：

* 此操作旨在将本地 Git 与 Github 账号建立连接，在本地创建 `ssh key` 来创建连接，生成公钥、私钥，复制粘贴公钥到 Github上，`$cat id_rsa.pub` 显示出公钥复制粘贴到 Github 上，`头像-> setting-> SSH and GPG keys-> new ssh key`，这样就建立好了远程仓库和本地仓库的连接。
* 存在远程仓库，如何与本地建立连接：克隆该仓库的 ssh 地址。
* 存在本地项目，如何与远程仓库建立连接：初始化本地仓库 ---> `git remote add + 远程仓库ssh地址`

#### 2. 远程仓库操作

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
git remote set-url https：//xx                 // 重置，如果之前弄错了
```

注：
* origin为远程仓库的名字，可以更改但不推荐。
* [clone下载速度很慢怎么办？](https://jscode.me/t/topic/789/2)
* Github国内的替代品有：
	* [腾讯云开发者平台](https://dev.tencent.com/)
	* [码云Gitee](https://gitee.com/)
	* [Gitlab](https://about.gitlab.com/)



### 三、Github 团队协作

#### 1. 在拥有对方仓库 `读 / 写` 权限的情况下

* 将对方仓库 `clone` 到本地进行修改，在对方仓库在新建 `branch` 进行`commit、push` 操作，随后提交 `pull request` ，备注修改内容及意见。

#### 2. 在没有拥有对方仓库 `读 / 写` 权限的情况下

* 将对方仓库 `fork` 到自己仓库，再 `clone` 到本地进行修改，`commit、push` 之后到对方仓库提交 `pull request` ，备注修改内容及意见。



###  四、实用 Git 技巧

* 快捷键设置

``` shell
vi ~/.bash_profile
alias ga="git add"
alias gc="git commit -v"
alias gp="git push"
alias gst="git status"
alias gco="git checkout"
alias glog="git log --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s%Cgreen(%cr) (bold blue)<%an>%Creset' --abbrev-commit -- | less"  // 美化git log输出
source ~/.bash_profile                                    // 让它生效
```

注：
* `git rebase` 可以美化提交历史，现阶段用不到，略。

