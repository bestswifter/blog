# Cocoapods 源码调试

## 综述

对于任何大型工程来说，硬着头皮直接看源码，往往不是最高效的选择。如果能直接从源码运行起来，并且打上断点，可以说就已经解决 80% 的问题了。
因为函数间的调用关系已经不再是瓶颈，最多就是有一些语法上的规则不了解，可能会影响阅读了。

在开始调试之前，也参考了一些网上的资料：

* [使用 VSCode debug CocoaPods 源码和插件](https://github.com/X140Yu/debug_cocoapods_plugins_in_vscode/blob/master/duwo.md)：我司大佬同事写的文章，但是我个人还是不太喜欢用 VSCode。
尤其是这类脚本语言，JetBrains 家的 IDE 总感觉更加专业一些。而且插件的调试流程不太自然，过于 hack 了。
* [Cocoapods 插件调试环境配置](http://dreamtracer.top/cocoapods-cha-jian-diao-shi-huan-jing-pei-zhi/)：和这个类似的还有几篇文章，但基本上大同小异，我没有成功跑起来，里面的逻辑也感觉比较奇怪。

本文主要介绍，如何**基于 RubyMine** 这个 IDE，创建一个**隔离的环境**，来调试 Cocoapods 和相关的插件。

## 背景知识介绍

在折腾源码调试之前，我对 Ruby 的工具链并不熟悉（现在也是一只半解）。但是这部分知识的确实，会比较影响后续排查问题，以及对整体流程的理解。所以有必要提前解释下。

我会尽量从基础到实际，用简单的语言来描述下相关工具的作用。不少概念参考自御姐的文章：[为什么我们要使用 RVM / Bundler ？](https://juejin.im/post/5c1fb3696fb9a049af6d4132)

### RVM

rvm 的全程是 `ruby version manager`，用来在命令行下，提供多 Ruby 版本的管理和切换能力。它不仅可以管理多个 ruby 版本， 也可以用来替换系统自带的 Ruby。那个 Ruby 配置有很多问题，比如最典型的，
`gem install` 安装依赖报错，没有权限。

安装：

```bash
# 可能需要用 gpg 命令先进行验证，根据命令行提示来操作一般都没问题
curl -sSL https://get.rvm.io | bash
```

使用 `2.6.2` 版本的 ruby

```bash
rvm install '2.6.2'
rvm use 2.6.2
```

### gem

这是 Ruby 的包管理工具，类似于 iOS  `cocoapods`，JS 的 `npm`，macOS 的 `homebrew`

当我们使用 rvm 来管理 ruby 版本后，和 `ruby` 命令一样，`gem` 命令也不再是系统自带的，而是由 `rvm` 统一提供了。

以通过 rvm 安装的 ruby 2.6.2 为例，这个可执行的 `gem` 二进制，位于目录 `$HOME/.rvm/rubies/ruby-2.6.2/bin/gem` 下

可以通过 `gem env` 命令来查看更详细的配置：

![](../pictures/cocoapods-debug/gem-env.png)

如果需要安装 Ruby 的包，可以通过下面命令：

```bash
gem install bundler
```

大部分 gem 包的安装路径都在上图中的第二个路径，即 `$HOME/.rvm/rubies/ruby-2.6.2/lib/ruby/gems/2.6.0` 下。也有少数被安装在第一个路径 `$HEOME/.rvm/gems/ruby-2.6.2` 下。不过都不影响使用（有了解的同学可以指教一下这俩路径的区别）。

### bundler

这是一个特殊的 gem 包，安装后会得到一个叫做 `bundle` 的可执行文件。主要是用来管理所有 `gem` 包的版本的。

举个例子，如果没有 `bundler` 这个包，每个项目都需要手动执行多个 `gem install` 命令。由于语义化版本号的范围依赖问题，不同时候执行 `gem install` 命令得到的结果可能是不一样的（比如开发者又发布了更新的版本，默认就用上了）。

`bundler` 的解决方案是，它定义了一个 `Gemfile` 文件，用来描述一个项目所有的 `gem` 包依赖情况。

然后通过 `bundle install` 命令一件安装，并且生成 `Gemfile.lock` 文件来锁定版本号。由于本文的读者至少是对 Cocoapods 有一定了解的开发者，这个文件和机制的作用，就不需要再解释了。

`bundler` 还提供了 `bundle exec` 命令，来确保依赖的准确性。

上文介绍了，默认情况下 `gem install` 安装的 `gem` 包，都会存储到一个全局统一的路径下。如果我们安装了同一个 `gem` 包的不同版本， `bundle exec` 命令就可以根据 Gemfile 中的依赖，自动选择正确版本的 `gem` 包了。

> 对于新人来说，bundler 这个库确实比较怪，一句话总结就是：安装了 `bundler` 这个包后，可以用 `bundle install` 命令来读取 `Gemfile` 文件中定义的 `gem` 包依赖，自动批量安装，并且生成 `Gemfile.lock` 文件锁定版本号

## 源码调试

### 原理

经过一番摸(cai)索(keng)，我整理了一下使用 RubyMine 调试的原理。由于都是经验性的结论，就不上分析过程了。

要想调试 `pod install` 等命令的流程，我们至少需要两个目录：

* 源码目录：这里面要存放 cocoapods 的源码，如果要调试插件，插件的源码也要在这里。
* iOS Demo 工程目录：这里面要有 Demo 工程和用于测试的 Podfile，并且使用上面源码版本的 Pod

对于 RubyMine 这个 IDE 来说，它的调试依赖于 `ruby-debug-ide` 这个 IDE，并且需要进入到 Demo 工程目录去执行源码版本的 Pod。

理解这个原理非常重要，因为它看起来会带来一些矛盾：

* 我们日常需要基于源码目录进行开发， 查看函数的定义，写代码等等。Gemfile 也需要在这个工程下配置，这样才能源码依赖和调试插件。
* Demo 工程理论上应该和源码目录完全解耦，因为一份源码工程完全可以对应多个 Demo 工程，不能叫接收两者耦合
* 实际的 `Pod Install` 是发生在 Demo 工程目录下的，需要在这里引用到 Cocoapods 源码，并且正确的配置相关依赖

上述三个需求是预期达到的目的，再基于之前的原理分析，初步可以形成一个思路：“主体配置放在源码目录中进行日常开发和管理，Demo 工程中通过软连接的方式获取到 Gemfile 配置”。

下面分别介绍 Cocoapods 源码目录和 Demo 工程目录的配置。

### Cocoapods 源码目录配置

首先创建一个空的目录，然后把我们要调试的代码下载下来：

```bash
git clone https://github.com/CocoaPods/CocoaPods.git
git clone https://github.com/CocoaPods/Core.git cocoapods-core
git clone https://github.com/leavez/cocoapods-binary.git
```

这里把 cocoapods-core 模块也已源码的方式引入，原因有两个：

1. cocoapods 工程是由若干个子模块共同组合的，尝试一下 cocoapods-core 这个子模块的调试体验
2. 目前(2020.7) 最新的 cocoapods 仓库中，依赖于 cocoapods-core 模块提供的 `post_integrate` 钩子函数。但是 这个函数的实现，并没有跟随 `cocoapods-core` 的最新版本发布（也停留在 master 分支上）。导致无法正常运行。