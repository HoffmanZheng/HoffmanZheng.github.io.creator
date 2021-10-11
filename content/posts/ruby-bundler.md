---
title: "Ruby：版本管理 RVM、Gem 与 Bundler"
author: "Chenghao Zheng"
tags: ["Ruby"]
categories: ["Study notes"]
date: 2021-10-10T13:19:47+01:00
draft: false
---

本篇介绍在 Ruby 项目中版本及包管理的工程实践，包括使用 [RubyGems](https://rubygems.org/) 管理 Ruby 的组件，使用 [Bundler](https://bundler.io/) 来解决项目中 gem 组件的依赖问题，使用 [RVM](https://rvm.io/) 管理不同版本的 Ruby 环境等。

推荐阅读：

1. [Bundler 到底是怎么工作的(暨 Ruby 依赖管理历史回顾)](https://ruby-china.org/topics/28453)
2. [Ruby Gemfile 详解](https://ruby-china.org/topics/26655)

### RubyGems

Ruby 主要通过 `require` 函数来引入外部的库文件，参数可以传文件名，相对路径或是绝对路径。当参数是文件名时，Ruby 会去 `$LOAD_PATH` 这个全局变量定义的路径中搜索库文件。

```powershell
$ ruby -e "puts $:" 
/usr/local/lib/ruby/site_ruby/2.7.0
/usr/local/lib/ruby/site_ruby/2.7.0/x86_64-darwin20
/usr/local/lib/ruby/site_ruby
/usr/local/lib/ruby/vendor_ruby/2.7.0
/usr/local/lib/ruby/vendor_ruby/2.7.0/x86_64-darwin20
/usr/local/lib/ruby/vendor_ruby
/usr/local/Cellar/ruby@2.7/2.7.4/lib/ruby/2.7.0
/usr/local/Cellar/ruby@2.7/2.7.4/lib/ruby/2.7.0/x86_64-darwin20
```

$LOAD_PATH 是一个数组变量，里面存放依赖路径字符串，可以看到其中的目录可以分成三大类：

* site_ruby 默认优先级最高，安装本机相关库
* vendor_ruby 操作系统供应商进行定制用的，一般为空
* 2.7.4 ruby 标准库目录，比如 date, csv 库

require 会按照 $LOAD_PATH 数组里面的路径顺序进行查找，找到了就不继续往下找了。

[RubyGems](https://rubygems.org/) 是用来寻找并管理 Ruby 组件的工具，让你可以轻松下载别人的代码，它重写了 require 函数的实现，并将 gem 安装到和 site_ruby 平级的 `gems` 目录下。gem 工具允许你用一个单一命令完成下载以及安装，允许你一键卸载，并且中心化管理所有安装了的库。

```powershell
gem install rails -v 4.1
gem uninstall
gem list rails
```

但 RubyGems 也有没有解决的问题，一个大问题就是没有依赖的版本控制，并且大多数 gem 都被安装进的同一个路径下，如果系统中存在多个需要 gem 的 Ruby 项目，又该如何对依赖的 gem 版本进行分别管理呢？

### Bundler

Bundler 的出现修复了 RubyGems 没有解决的问题，它让项目可以根据定义来使用 gem，并且在安装 gem 时就进行版本冲突的解析。Ruby 的开发者只需要列出他所需要的 Gem，然后 Bundler 就会找出合适的版本让它们在一起工作，并且把一个可行解（但不一定是最优解）放入 Gemfile.lock。这个文件保证了共享代码或者部署到服务器时能够安装到正确的依赖版本。

Bundler 的工作原理，See: [Bundler's Purpose and Rationale](https://bundler.io/rationale.html)。在应用根目录 `Gemfile` 文件里声明依赖后，Bundler 会去 source 指定的 `https://rubygems.org` 上寻找 gem

```powershell
source 'https://rubygems.org'

gem 'rails', '4.1.0.rc2'
gem 'rack-cache'
gem 'nokogiri', '~> 1.6.1'
```

项目第一次安装依赖时可以执行 `bundle install --path=vendor/bundle` 把 gem 安装到项目的 vendor/bundle 目录下，再在 git 中忽略此目录，这样做就不会因为多个项目安装 gem 到系统目录，而导致系统里的 gem 冲突。

因为在 Gemfile 中声明的依赖有它们自己的依赖，所以运行 bundle install 会安装相当多的 gem。如果任何需要的 gem 已经被安装了，bundler 会直接使用它们。在所有需要的 gem 被成功安装后，bundler 会写一个所有这些 gem 和它们的版本号的快照到 `Gemfile.lock` 中。

如果开发的是个 Rails 应用，默认就有可以可以运行 bundler 的代码了。对于其他的应用（比如基于 Sinatra 的应用），需要在引用任何 gem 之前配置一下 bundler。比如在 `require 'sinatra'` 的那个文件的第一行加入以下代码：

```powershell
require 'rubygems'               // Ruby 1.8 以后的版本不再需要这句
require 'bundle/setup'
```

这样 bundler 就能将 gem 的地址加入到 `$LOAD_PATH` 中，以此来让 require 正常工作。

Gemfile 和 Gemfile.lock 应该被一起放到 **版本管理** 中，这样版本库中就有了应用最后一次确定能正常工作时所有的 gem 及版本号的记录。当在别的机器上获取应用代码并执行 bundle install 时，bundler 会找到 Gemfile.lock，跳过解决依赖的步骤并按照之前记录的依赖版本号获取 gem。仅在 Gemfile 确切地指定依赖的第三方版本并 **不能保证** 应用使用正确的依赖版本，因为 gem 通常给它们自己的依赖声明一个版本号的范围。 

Bundle install 会 **保守** 地执行依赖升级，不会更新在 Gemfile 中没有显式更改的 gem（或者它们的依赖）。举个例子，rails 依赖了 actionpack，actionpack 又依赖了 rack，与此同时 rack-cache 也依赖了 rack，如果在更新 rails 的同时也更新了依赖的 rack，没有得到更新的 rack-cache 就可能与新版的 rack 出现不可预见的兼容问题。对此，如果 Gemfile 中的 rack-cache 没有被修改，bundler 就会把它和它的依赖 (rack) 当成一个不可修改的整体。

可以使用 `bundle update` 命令（不推荐），在不修改 Gemfile 的情况下更新 gem，但这个命令会从头开始解决依赖并忽略掉 Gemfile.lock，需要做好全面的测试和 `git reset --hard` 的准备。

```powershell
bundle update rack-cache        // 更新 rack-cache 到 Gemfile 里允许的最新版本
bundle update                   // 升级所有 Gemfile 里的 gem 到最新能用的版本
```

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









