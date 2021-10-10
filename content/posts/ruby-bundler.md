---
title: "Ruby：版本管理 RVM、Gem 与 Bundler"
author: "Chenghao Zheng"
tags: ["Ruby"]
categories: ["Study notes"]
date: 2021-10-10T13:19:47+01:00
draft: false
---

本篇介绍在 Ruby 项目中版本及包管理的工程实践，包括使用 [Gem](https://rubygems.org/) 管理 Ruby 的组件，使用 [Bundler](https://bundler.io/) 是用来解决项目中 Gem 组件的依赖问题，使用 [RVM](https://rvm.io/) 管理不同版本的 Ruby 环境等。

推荐阅读：

1. [Bundler 到底是怎么工作的(暨 Ruby 依赖管理历史回顾)](https://ruby-china.org/topics/28453)
2. [Ruby Gemfile 详解](https://ruby-china.org/topics/26655)

### RubyGems

[RubyGems](https://rubygems.org/) 是用来寻找并管理 Ruby 组件的工具，让你可以轻松下载别人的代码。gem 工具允许你用一个单一命令完成下载以及安装，允许你一键卸载，并且中心化管理所有安装了的库。

```powershell
gem install -v 4.1
gem uninstall
gem list rails
```

但 RubyGems 也有没有解决的问题，比如存在多个需要 gem 的项目，该如何对依赖的 gem 版本进行分别管理？

### Bundler

Bundler 的出现修复了 gem 没有解决的问题，它让项目可以根据定义来使用 gem，并且在安装 gem 时就进行版本冲突的解析。Ruby 的开发者只需要列出他所需要的 Gem，然后 Bundler 就会找出合适的版本让它们在一起工作，并且把一个可行解（但不一定是最优解）放入 Gemfile.lock。这个文件保证了共享代码或者部署到服务器时能够安装到正确的依赖版本。

项目第一次安装依赖时可以执行 `bundle install --path=vendor/bundle` 把 gem 安装到项目的 vendor/bundle 目录下，再在 git 中忽略此目录，这样做就不会因为多个项目安装 gem 到系统目录，而导致系统里的 gem 冲突。

### Ruby enVironment Manager

RVM 支持管理多个 Ruby 应用环境并且支持切换。对 Ruby 版本管理的方法如下（See: [The Basics of RVM](https://rvm.io/rvm/basics)）：

```powershell
$ rvm install 2.7.4             // 安装并使用指定版本
$ rvm list                      // 列举 RVM 安装过的 Ruby 版本  
# =* - current && default
 * ruby-2.6.3 [ x86_64 ]        #  * - default
=> ruby-2.7.4 [ x86_64 ]        # => - current
$ ruby -v                       // 当前使用的 Ruby 版本
$ which ruby                    // 当前使用的 Ruby 的路径
/Users/hoffman.zheng/.rvm/rubies/ruby-2.7.4/bin/ruby   // 注意放在~/.rvm 目录下
$ rvm --default use 2.7.4       // 设置默认的 Ruby 版本
$ rvm use 2.6.3                 // 设置当前的 Ruby 版本
```

RVM 会隔离当前操作系统中已经安装的 Ruby 版本，如需切换回系统的 Ruby，可以让 RVM 撤销已经应用的环境更改：

```powershell
$ rvm use system                // 切换回系统的 Ruby
$ which ruby
/usr/local/opt/ruby@2.7/bin/ruby     // 切换回的系统 Ruby 的路径
```

RVM 会为每个版本的 Ruby 创建一个完全隔离的 Gem 目录，此外还可以根据项目/应用将 Gem 依赖们进一步分开（See: [Named Gem Sets](https://rvm.io/gemsets/basics)）：

```powershell
$ which ruby                    // 当前使用的 Ruby 的路径
/Users/hoffman.zheng/.rvm/rubies/ruby-2.7.4/bin/ruby   // 注意放在~/.rvm 目录下
$ rvm gemdir                    // 当前版本 Ruby 的 Gem 目录
/Users/hoffman.zheng/.rvm/gems/ruby-2.7.4
$ rvm 2.6.3@testing --create    // 为 2.6.3@test 创建一个隔离的 gems 文件夹
$ rvm gemdir
/Users/hoffman.zheng/.rvm/gems/ruby-2.6.3@test
$ rvm gemset create alias1 alias2     // 为当前版本 Ruby 创建两个 gems 环境
$ rvm 2.6.3@alias1              // 切换使用的 gems 环境
```

可以在文件资源管理器中看到，RVM 为 2.6.3 版本的 Ruby 又创建了一个单独的 gems 文件夹（如果根本没有使用 RVM 的 gemset，会从 default 目录获取 gems；一个具名的 gemset 将会从 global 中继承 gems，或者说 global 文件夹允许用户共享 gems）：

![](/images/ruby-namedGemsets.png)

如果仍对 RVM 的工作流不熟悉，可以参考下官方给的 workflow：[Examples of using RVM](https://rvm.io/workflow/examples)









