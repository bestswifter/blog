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

我会尽量从基础到实际，用简单的语言来描述下相关工具的作用

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




