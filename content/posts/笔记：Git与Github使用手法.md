---
title: "笔记：Git与Github使用手法"
date: 2019-12-26T21:49:47+01:00
draft: false
---

# Git 本地仓库（版本管理）
* 应用场景一：

```
甲方：我这个背景颜色要五彩斑斓的黑
甲方：我还是想要土耳其沙滩上空蓝天的那种蓝色，你再改下
甲方：我还是觉得原先设计的那种颜色比较好看，改回去把
我：好的（MMP，不早说，我现在已经没法撤销了！）
```

* 应用场景二：

```
老板：给我做一个页面
我：花了几天，做出来了，请过目
老板：不够醒目，再改改
老板娘：我觉得挺好，要是背景能做成五彩斑斓的黑就好了
我：好嘞，我做两份你们对比一下（MMP，你们俩就不能统一意见吗！）
```

* 应用场景三：

```
老板：昨天夜里我和老板娘达成一致了，两个版本都要，我的放在上面，老板娘的在下面
我：好的，我合并一下 (￣▽￣)"
```

0. 安装 Git

[参考伯克利课程 CS61b 中的教程安装 Git](https://sp18.datastructur.es/materials/lab/lab1setup/windows.html)

1. 配置 Git

```shell
git config --global user.name + 你的英文名
git config --global user.email + 你的邮箱
git config --global push.default simple
git config --global core.quotepath false   // 不会对0×80以上的字符进行quote，中文显示正常。
git config --global core.editor "code --wait"
git config --global core.autocrlf input
git config --global --list                 // 查看当前配置
vi ~/.gitconfig                            // 编辑配置文件
```

2. Git 本地仓库（应用场景一）

```shell
cd + 目标路径
git init                              // 初始化，当前目录创建 .git 文件夹
git add + 文件 / 文件夹 / .            // 选择要提交的内容
git commit -m "版本信息 / 更新日志"     // 写更新日志，提交代码快照
git log 
git reflog                            // 查看所有备份记录
git status
git reset --hard XXXXXX               // 版本回溯 写commit号前6位
```

注：
* 可以在目录下新建个 `.gitignore` 文件，把不需要 `git` 管理的文件/文件夹路径写进去。常见的有`.idea` `.vscode` 等。
* 在使用 `git reset` 回溯版本之前，一定要确保当前代码已经 `commit` 过了，因为这个操作会使当前没有 `commit` 的变动消失。

3. Git 分支管理（应用场景二）

```shell
git branch                               // 查看现在存在的分支
git branch + newBranchName               // 新建分支，把当前代码快照复制到该分支
git checkout branchName / master         // 切换分支
history                                  // 查看所有操作记录
```

4. Git 合并分支（应用场景三）

```
git checkout branchName / master        // 进入要保留的分支
git merge brachName                     // 将brach合并至当前分支
git add --> git commit
git branch -d branchName                // 删除分支
```

注：
* 合并分支时会检查代码是否有冲突，有冲突的话需要解决冲突，搜索 `====` 选择要保留的代码，再次合并。

5. 其他操作

```shell
start .                               // 在资源管理器打开当前文件夹
subl (文件 / 文件夹)                   // 快速打开sublime
git rm -r --cached + filename         // 删除add的路径
git add fileName --> git stash        // 暂时不像提交代码（通灵术）
git stash pop
```

# Github 远程仓库（云端备份）

* 应用场景

```
我需要同时在公司和家里写代码
笔记本被女朋友的奶茶泡坏了
不小心运行了 rm -rf/
在云端实时备份代码就能解决以上所有问题呢 (ง •̀-•́)ง
```

1. SSH key 验证身份（关联Github账号）

```shell
ssh-keygen -t rsa -b 4096 -C 你的邮箱           // 生成公钥文件
cat ~/.ssh/id_rsa.pub                          // 打开公钥文件，复制公钥到网页
ssh -T git@github.com                          // 测试配对成功
```

2. 远程仓库操作

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
- origin为远程仓库的名字，可以更改但不推荐。
- [clone下载速度很慢怎么办？](https://jscode.me/t/topic/789/2)
- Github国内的替代品有：
    - [腾讯云开发者平台](https://dev.tencent.com/)
    - [码云Gitee](https://gitee.com/)
    - [Gitlab](https://about.gitlab.com/)

#  实用Git技巧

* 快捷键设置

``` shell
vi ~/.bashrc
alias ga="git add"
alias gc="git commit -v"
alias gp="git push"
alias gst="git status"
alias gco="git checkout"
alias glog="git log --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s%Cgreen(%cr) (bold blue)<%an>%Creset' --abbrev-commit -- | less"  // 美化git log输出
source ~/.bashrc                                     // 让它生效
```

注：
* `git rebase` 可以美化提交历史，现阶段用不到，略。

