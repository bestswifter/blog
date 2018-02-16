> 本文是视频直播的文字整理，录像可以在：[优酷](http://v.youku.com/v_show/id_XMTU4Nzg3NjAwMA==.html) 上看到

关于 Mac 工作效率的文章一直层出不穷，然而并非所有内容都适合程序员，比如某些 Unix 命令，其实使用频率非常低。作为一名初级 iOS 程序员，我尝试着和大家分享一些能够切实提高我们开发效率的小技巧。

我是无鼠标主义者，任何需要鼠标的操作在我看来都是极为低效的。Mac 的触摸板非常好用，但是我依然在尝试避免使用触摸板。因为双手保持在键盘区域更适合编程。虽然触摸板不可能被避免（比如浏览网页），但我希望至少在 Xcode 中不使用它。

所以，本文会和大家分享一些系统级快捷键，Xcode、Chrome、iTerm 等应用中的快捷键，以及常用的工具，比如 Vim 和 Git 的使用。这里面除了 Xcode，其他都是通用的，如果你不是 iOS 开发者，建议自行查阅相关 IDE 的快捷键。

## 综述

一部分人可能认为，快捷键用起来很别扭，还不如自己用触摸板（鼠标）来得方便。然而你应该意识到，使用触摸板的效率是有上限的，当你熟悉快捷键后，速度远比现在快得多。

这一点，在学习 vim 时尤其重要。你不应该关注完成一个命令需要多久，而应该关注需要多少个按键，你可以认为在形成肌肉记忆后，按键的思考时间为0。所以我们得出一个结论：

总时间 = 按键数 * 一个常数（表示单次按键时间）。

因此，评价 vim 中一个操作的优劣，通常用**高尔夫分数**来表示，它表示完整这个操作需要几次按键。

**但是！！！快捷键是提高效率的手段，但它不会提高代码质量。既要坚持学习，也要适可而止，万万不可主次颠倒。**

关键不在于你学会了多少快捷键，而是你有多少工作是可以通过快捷键来完成的，目的在于提高效率，仅此而已。

一种很强大，通用的的方法是 设置->键盘->快捷键->应用快捷键 然后精确匹配应用中的快捷键名，这个通常需要配合 CheatSheet 来实现。当你觉得某个快捷键不好用的时候，也可以通过这种方式去修改。

![应用内快捷键替换](http://images.bestswifter.com/shortcut/1@2x.png)

在设置快捷键时，需要避免全局快捷键和应用快捷键冲突，同时也要注意一些常用操作在多个应用内保持统一。

我建议将 Caps Lock 与 Ctrl 键对调，因为大小写切换键的使用频率非常低，而 Ctrl 的使用频率显然高于他，因此有必要将大小写切换键放到最不容易触碰到的地方。

下面我会介绍一些我常用的快捷键，它们大部分是系统自带的，也有少部分是我自己定义的。

## 入门

1. 绝大多数应用的 preference 页面都是通过 `Command + ,` 打开的。
2. 剪切，复制，粘贴，撤销，重做，光标移动到行首和行尾，这些基础操作必须掌握。

## Snap

相信很多人都有这样的烦恼：如果应用不全屏，那么桌面上显示的窗口太多，每个窗口的显示内容不够多。如果应用全屏，那么切换应用是很麻烦的。要么用 `Command + Tab`，要么手势滑动，但无论哪一种，时间复杂度都是 O(n)。有没有 O(1) 的方法呢？答案是使用神器：**snap**

我主要是以应用首字母或者关键字母作为标识，配合 `Command + Shift` 前缀：

1. **Xcode：`J`**
2. **Chrome：`K`**
3. **iTerm：`L`**
4. Markdown 相关：`M`
5. QQ：`Y`
6. 微信：`U`
7. SoureceTree：`S`
8. MacVim：`V`
9. Evernote：`E`
10. **Dock：`1/2/3/4`：因工作需要，我常用的是备忘录，邮件，日历，设置**
11. `;` 这个键我没有启用，但它实际上是一个非常方便的快捷键。

Dock 栏应用的选择需要一定的权衡。显然最快的方式是只按 `Command`，但是这种全局快捷键会导致大量冲突。而 `Controll` 和 `Option` 键又非常难以触摸，所以我选择了 `Command + Shift` 作为所有应用的快捷键前缀。

**注意避免字母 `o` 和 `f`，它们在 Xcode 中有特殊的用处。**

![Snap](http://images.bestswifter.com/shortcut/2.png)

## Xcode 快捷键

编译、运行，Instruments，单元测试，暂停这些基本操作就不解释了。我把一些自认为比较有用的命令加粗表示：

### 文件编辑

1. `Command + [` 和 `Command + ]` 左右缩进
2. **`Command + Option + [` 和 `Command + Option + ]` 当前行上下移动**
3. `Command + Option + Left/Right` 折叠、展开当前代码段

### 文件跳转

1. **`Command + Control + Up/Down` .h 和 .m 文件切换**
2. **`Command + Control + Left/Right` 浏览历史切换**
3. `Command + Control + j` 跳转到定义处
3. `Command + Option + j` 跳转到目录搜索
4. `Command + 1/2/3/4/5` 跳转到左侧不同的栏目
5. **`Comannd + Shift + o` 文件搜索**

### 搜索

1. `Comannd + Shift + f` 全局搜索
2. `Command + e` 搜索当前选中单词
3. `Command + g` 搜索下一个

### tab

1. `Command + t` 新建一个 tab
2. `Command + w` 关闭当前 tab
3. **`Command + Shift + [` 和 `Command + Shift + ]` 左右切换 tab**

### Scheme

1. `Command + shift + ,` 编辑 scheme，选择 debug 或 release

### 调试

`F6`：跳到下一条指令
`F7`：跳进下一条指令（它会跳进内部函数，具体效果自测）
`Control + Command + y` 继续运行

### 其他

1. `Command + k` 删除 Console 中的内容
2. `Command + d` 打开/关闭 控制台（修改系统快捷键：Show/Hide Debug Area）

获得更全面的快捷键介绍，请参考：[这篇文章](http://stackoverflow.com/questions/10296138/xcode-debug-shortcuts)

## Vim 常用快捷键

入门指南：[简明 Vim 练级攻略](http://coolshell.cn/articles/5426.html)
在我的 git 上有一份 Vim 的配置，先[下载](https://github.com/bestswifter/.vim)到 `~/` 目录下，然后建立软连接：

```bash
rm .vimrc
ln -s .vim/.vimrc .vimrc
```

推荐一个 Mac 上的 Vim 软件：MacVim，它比在终端中看 Vim 更好一些。打开 MacVim 后，输入以下命令安装插件：

```vim
:BundleInstall
```

### 进入输入模式

1. `i` 在光标前面进入输入模式，`a` 在光标后面进入输入模式
2. `I` 在行首进入输入模式，`A` 在行尾进入输入模式
3. `o` 在下一行行首进入输入模式，`O` 在上一行行首进入输入模式

### 文本操作

1. `yy` 复制当前行，`dd` 剪切当前行，`p` 复制。注意这里用的都是 Vim 自带的剪贴板。
2. `U` 撤销，**`Ctrl + r` 重做
3. `x` 删除光标所在的字母
4. `cae` 或 `bce` 删除当前光标所在的单词，并进入编辑模式
5. `数字+命令` 重复命令 n 次，比如 `3dd`

### 光标移动

1. `^` 到本行开头，`$` 到本行末尾
2. `:111` 或 `111G` 跳转到 111 行，`gg` 第一行，`G` 最后一行。
3. `e` 移动到本单词的结尾， `w` 移动到下一个单词的开头。
4. `%` 匹配当前光标所在的括号（小括号，中括号，大括号）
5. `*` 查找与光标所在单词相同的下一个单词
6. `f + 字母` 跳转到字母第一次出现的位置，`2fb` 跳转到字母 b 第二次出现的位置
7. `t + 字母` 跳转到字母第一次出现的前一个位置，`3ta` 跳转到字母 a 第三次出现的前一个位置
8. f 和 t 换成大写，表示反方向移动查找。`dt + 字母` 表示删除字母前的所有内容。

### 举一反三

1. `<start position><command><end position>` 比如 `0y$`，从行首复制到行尾，`ye` 表示
从当前位置复制到本单词结尾。
2. `<action>a<object>` 或 `<action>i<object>` 
	
	`action` 可以是任何的命令，比如 `d`，`y`，`v` 等
	`object` 可以是 `w` 单词，`p` 段落，或者是一个具体的字母
	`a` 和 `i` 的区别在于 `i` 表示 inner，只作用于内部，不含两端。
	
	思考一下，有多少种方法可以删除光标当前所在单词？
	
	答案：`diw`，`daw`，`caw`，`ciw`，`bce`，`bde`。
	
	思考一下他们的原理，后两者不太推荐（有可能跳到前一个单词）。
	
	如果是选中当前单词呢？

除了以上基本语法，我还在整理一套 《Vim 基础练习题》，等完成之后会与大家分享。

### 实战

1. 给多行添加注释：

	1. `v`：进入可视状态
	2. `nj`: 向下选择n行， 或者输入 `Shift ]` 跳到段尾
	3. `Command + /` 添加注释

2. 在 MacVim 中，`git blame` 无比清晰：

![Snap](http://images.bestswifter.com/shortcut/3.png)

## Chrome

1. **`Command + l` 焦点移动到地址栏**
2. **`Shift + Option + Delete/Left`** 向左删除/选中一个单词（可以自定义为 `Ctrl-w`）
3. `Command + y` 搜索历史
4. **`Command + 数字` 快速切换 tab**
5. `Command + shift + []` 左右切换 tab
6. `Command + t/w` 新建/关闭 tab
7. `Command + e/g` 搜索选中，前往下一个，或者用 `Command + f` 和回车。

可以看到，Chrome 中涉及到 tab 的操作应该与 Xcode 尽量保持一致。

## iTerm2

1. **`Ctrl w` 删除前一个单词**
2. `Command + r` 清除屏幕上的内容
3. `Command + t/w` 打开/关闭 tab
4. **`Command + 数字` 切换到第 n 个 tab**
5. `双击` 选中一个单词，自动复制

iTerm 可以通过 `Command + shift + []` 来左右切换 tab，也可以通过 `Command + Left/Right` 切换，后者其实是多余的，而且不符合习惯。

所以参考 [这篇文章](http://www.michael-noll.com/blog/2007/01/04/word-movement-shortcuts-for-iterm-on-mac-os-x/) 或者自行查阅 Google，在 Preference->Keys->Global Shortcut Keys 中，设置好 `Command` 加上左右键，和删除键的对应操作。

## Git

[git](http://gitbook.liuhui998.com/index.html) 的本质是对指针的操作。

掌握git的 `add`、`commit`、`stash`、`pull`、`fetch` 这些基本操作

理解什么是本地仓库，什么是远程仓库，理解多人开发时的 `merge` 和 `conflict` 的概念

掌握分支的使用，掌握 `checkout` 命令的使用

熟练掌握 [`git rebase`](http://gitbook.liuhui998.com/4_2.html) 操作，包括 [`git rebase -i`](http://gitbook.liuhui998.com/4_3.html) 和 `git rebase --onto`，掌握一种 git 工作流

## Oh my zsh

首先[下载 oh-my-zsh 的配置](https://github.com/bestswifter/.sys.config)到 `~/` 目录下，然后在命令行中执行以下操作：

```bash
rm .zshrc
ln -s .sys.config/.zshrc .zshrc
```

然后重启 iTerm。你可以根据自己的喜好，前往 `~/.sys.config/setting/git.zh` 配置 git 命令的别名，比如；

```bash
alias gcm='git commit -m'
alias gignore='git update-index --assume-unchanged'
alias gpush='git push origin HEAD:dev;'
alias go='git checkout'
```

## More

据说 Alfred 是效率神器，鉴于我除了写代码，一般不怎么玩 mac，所以也就没有去了解。如果有更多好的快捷键和应用，欢迎与我交流。