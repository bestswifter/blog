# 背景和目的

接来下的这篇文章会介绍如何完成一个**“不可能”**的任务——通过**改一个数字**，破解掉Paw这个收费软件。

起初是在某位大神的博客里看到了Mac上一款非常好用的App，叫<font color=red>Paw</font>。Paw可以在Mac上模拟各种HTTP请求，可视化的管理HTTP Header、Parameters、Cookies等，还有一点非常出乎意料的功能是通过下载插件可以自动生成Swfit、OC、JS等多种语言的代码。

然而Paw巨贵（198软妹币），而且破解版不好搜。于是寻思着自己动手解决需求，于是倒霉的Paw成了实验对象。[先从这里下载原版app](https://luckymarmot.com/paw)

由于在此之前我毫无逆向工程方面的经验，在看别人的介绍时各种不懂，深受折磨，所以我尽量用简单、详细的语言描述本次从零开始破解app之旅。作为参考，我用了**大约七个小时**的时间完成了此次破解（大量的时间浪费在找工具以及学习使用工具上，后面可以看到破解这个事情本身并不难）。在文章的最后会给出破解版的下载地址。

由于水平有限，只是介绍了基本的逆向工程知识，算是自己的学习笔记，也希望向更多的和我一样还只是菜鸟的程序员科普一些逆向工程的基本知识，同时督促自己平时在Coding过程中的注意代码规范和安全。

## 知识储备

想要破解app，首先自己得开发过app，至少了解一些基本的命令行操作，源代码、汇编代码和二进制码的基本定义。如果这些基本要求有某一点不满足，那么整个过程会是非常痛苦的。

## 工具准备
破解Paw用到的工具主要有以下几个。

+ <font color=red>homebrew</font> —— 不知道这个的估计都不好意思说自己是用Mac的程序员。
+ [<font color=red>Hopper Disassembler</font>](http://pan.baidu.com/s/1bn94SDx) —— 反编译工具，根据可执行文件反编译出汇编码。
+ [<font color=red>Class-dump</font>](http://stevenygard.com/projects/class-dump/) —— 逆向工程的入门级工具，导出一个App的某些信息。
+ <font color = red>otx</font> —— 国外某位大神的介绍的一个工具，我也说不出明确的用处。通过`brew install --HEAD homebrew/head-only/otx`命令安装。
+ [<font color=red>Hex friend</font>](http://ridiculousfish.com/hexfiend/) —— 二进制文件编辑器，要用这个修改二进制文件。
+ <font color=red>gdb</font> —— 著名的调试器，用lldb也行。通过`brew install gdb`命令安装。

## Begin cracking

### 找到破解点

要破解App当然要明白自己为什么要破解它，它哪一点限制了我们，首先运行原版的Paw。可以看到如下界面:

![å¯åŠ¨ç•Œé¢](https://o8ouygf5v.qnssl.com/AppCrack/real-paw-welcome.png)

这个Welcome界面非常讨厌，由于它的存在，我们不能点击程序主界面。而想要关掉这个Welcome界面，只有两个方法，选择**Try Paw**按钮获得30天试用期或点击**Register License**按钮输入自己的License。

因此我们的目的以及非常明确了——**<font color = red>关闭这个Welcome页面</font>**

### 初探Paw

既然要破解这个App，免不了要去了解这个App的结构。现在我们手上只有在Applications文件夹下的Paw.app这一个文件。突破口在于**Paw.app/Contents/MacOs/Paw**这个可执行的二进制文件。我们以后的操作，绝大多数时候是与它打交道。在“**应用程序**”文件夹下，右键Paw，选择“**显示包内容**”就可以看到这个二进制文件了

这时候，第一个工具——<font color = "rgb(226,238,250)">`class-dump`</font>出场了。由于篇幅所限，我就不介绍这个工具的具体配置方法了。可以参考这篇文章

>	[<font color = red>使用class-dump导出其他应用头文件</font>](http://www.jianshu.com/p/6a6ce18f998e)

我们先用<font color = "rgb(226,238,250)">`class-dump`</font>导出Paw的头文件看看，在终端中执行命令：

<font color = "rgb(226,238,250)">`class-dump -H /Applications/PawReal.app/Contents/MacOS/Paw -o /Users/你的用户名/Desktop/classdump`</font>,

换上你的用户名，等运行结束之后，在桌面上可以看到一个叫<font color = "rgb(226,238,250)">`classdump`</font>的文件夹。不要被里面密密麻麻的文件吓到，这就是这个app所有的头文件了。

### 换位思考，变通思路

接下来怎么找我们需要的信息呢，要想一个一个看过去，即使头文件里面只有方法和变量的定义，也是不现实的。好在<font color = "rgb(226,238,250)">`class-dump`</font>还有别的功能。执行命令：

<font color = "rgb(226,238,250)">`class-dump -f license /Applications/PawReal.app/Contents/MacOS/Paw`</font>

可以找到头文件中所有和license有关的部分。

会什么要找license呢，这个就需要猜了。既然这个软件需要注册码，并且Welcome界面有一个**Register License**按钮，一定会有一部分代码是用来管理证书（License）相关的。让我们站在开发者的角度上想，如果要遵守命名规范，那么头文件中也许会有**License**关键字的身影。

当然，这只是猜想，如果针对**License**关键字的查找结果不理想的话，我们还可以换一些关键字，比如**Register**、**Validate**等。

不过好在我们通过<font color = "rgb(226,238,250)">`class-dump`</font>发现了一些线索，如图所示：

![Class-dump](https://o8ouygf5v.qnssl.com/AppCrack/class-dump-license.png)

在图中，我们发现了一个比较有价值的类:<font color = "rgb(226,238,250)">`LMWelcomeViewController`</font>

### 合理猜想，趁胜追击

发现<font color ="rgb(226,238,250)">`LMWelcomeViewController`</font>这个用来管理Welcome页面的类之后，我们打开头文件看看里面的函数。很“巧”地，里面有一组函数，都是以<font color ="rgb(226,238,250)">`showWelcomeWindow`</font>开头。直觉告诉我们，这个用来显示Welcome页面的方法，很有可能就是解决问题的关键。

故技重施，再看一看<font color ="rgb(226,238,250)">`showWelcomeWindow`</font>这个函数的信息。运行：

<font color = "rgb(226,238,250)">`class-dump -f showWelcome /Applications/PawCrack.app/Contents/MacOS/Paw`</font>

可以看到这样的结果：

![æœç´¢ç»“æžœ](https://o8ouygf5v.qnssl.com/AppCrack/class-dump-showWelcome.png)

这就基本上印证了之前的猜想:<font color = "rgb(226,238,250)">`LMApplicationDelegate.m`</font>中的代码在程序启动时执行，通过某种方式判断用户是否已注册，**如果没有的话**，就调用<font color ="rgb(226,238,250)">`showWelcomeWindow`</font>这个函数，同时把<font color ="rgb(226,238,250)">`LMWelcomeViewController`</font>类的实例对象作为参数，这个对象再执行自己的<font color ="rgb(226,238,250)">`showWelcomeWindow`</font>方法，**展示Welcome页面**

当然，这样的分析很可能是错的。因为判断是否注册这个逻辑并不一定在<font color = "rgb(226,238,250)">`LMApplicationDelegate`</font>中进行，也可以放在<font color ="rgb(226,238,250)">`LMWelcomeViewController`</font>里。但无论如何，注意到黑体字部分连起来也是一段话，整个过程其实是一个**“如果……，就……”**的逻辑。

记住这个逻辑，一会儿我们会根据这个逻辑做一些修改！

### 细看函数实现

我们已经知道<font color ="rgb(226,238,250)">`LMWelcomeViewController`</font>的<font color ="rgb(226,238,250)">`showWelcomeWindow`</font>方法可能是解决问题的关键，接下来我们就来看看这个方法到底是怎么实现的。

打开Hopper Disassembler，把**MacOS**文件夹下的**Paw**二进制文件拖入其中，开始分析。Hopper Disassembler可以根据二进制文件反汇编成汇编代码。刚打开的时候，这个软件是这个样子：

![Hopper Disassembler](https://o8ouygf5v.qnssl.com/AppCrack/Hopper-Disassembler-introduction.png)

在这个软件中看到的东西往往非常奇葩，和任何一种高级编程语言都不同。这种汇编语言给新手的阅读造成了极大的障碍，好在有一些注释，也可以生成伪代码，辅助我们阅读。我几乎不太能看懂汇编语言，所以尽量避免过多的研究他们。

在左边的Labels标签下搜索我们感兴趣的内容，比如刚刚说的<font color ="rgb(226,238,250)">`showWelcomeWindow`</font>方法。

可以分别看到在<font color ="rgb(226,238,250)">`LMWelcomeViewController`</font>和<font color = "rgb(226,238,250)">`LMApplicationDelegate`</font>中<font color ="rgb(226,238,250)">`showWelcomeWindow`</font>方法的实现：

![<font color = "rgb(226,238,250)">`LMApplicationDelegate`</font>](https://raw.githubusercontent.com/649395594/AppCrack/master/search-1.png)

![<font color ="rgb(226,238,250)">`LMWelcomeViewController`</font>](https://raw.githubusercontent.com/649395594/AppCrack/master/search-2.png)

<font color = "rgb(226,238,250)">`LMApplicationDelegate`</font>中的<font color ="rgb(226,238,250)">`showWelcomeWindow`</font>方法非常简单，根据绿色部分的注释可以猜到调用了参数的<font color ="rgb(226,238,250)">`showWelcomeWindow`</font>方法。或者我们可以选中这段汇编代码，点右上角的![](https://o8ouygf5v.qnssl.com/AppCrack/pseudo-icon.png)图标生成伪代码。

![åæ±‡ç¼–åŽçš„ä»£ç ](https://o8ouygf5v.qnssl.com/AppCrack/pseudo1.png)

<font color = "rgb(226,238,250)">`LMWelcomeViewController `</font>中的<font color ="rgb(226,238,250)">`showWelcomeWindow`</font>方法比较复杂。

之前的图片上可以看到两个汇编指令分别是：<font color = "rgb(226,238,250)">`je `</font>和<font color = "rgb(226,238,250)">`ret `</font>

<font color = "rgb(226,238,250)">`je`</font>是**"jump euqal"**的缩写，表示如果相等，则跳转到某个地址。所以我们可以在<font color = "rgb(226,238,250)">`je `</font>的上面一行看到<font color = "rgb(226,238,250)">`cmp `</font>指令。与<font color = "rgb(226,238,250)">`je `</font>相对应的就是<font color = "rgb(226,238,250)">`jne `</font>，表示**"jump not euqal"**

<font color = "rgb(226,238,250)">`ret`</font>顾名思义就是**return的缩写了**，表示函数在这里返回。

其实在这里我们已经可以大概了解这个</font>中的<font color ="rgb(226,238,250)">`showWelcomeWindow`</font>方法的实现了。进行了一个判断，如果城里就返回，否则就进行下面一段操作，而根据右侧绿色提示，我们看到了**“可怕”**的<font color ="rgb(226,238,250)">`showWindow`</font>方法，这个方法没有在头文件里面看到，估计就是一个 私有方法了。

如果不放心的话还可以生成伪代码看看：

![ä¼ªä»£ç ](https://o8ouygf5v.qnssl.com/AppCrack/pseudo2.png)

### 巧变逻辑

之前分析了整个Welcome页面出现的逻辑其实是一个**“如果……，就……”**的判断，那么要想破解，也很容易。方法有两个，要么判断条件不成立，要么改变执行语句。显然，对于不熟悉汇编和逆向工程的新手而言，让判断条件不容易更加简单一些。注意到<font color = "rgb(226,238,250)">`je`</font>指令之前有一个数字：**00000001000cdfaf**，它表示的是这条指令在虚拟内存空间中的地址。那么这个地址有什么用呢？

### 梳理思路

确实乍一看，获取指令的地址并没有用处。而且从开始到现在，一直在接触完全没接触过的东西，已经有点晕乎了。

梳理一下到目前为止的思路，我们从**license**关键字树藤摸瓜，找到了<font color ="rgb(226,238,250)">`showWelcomeWindow`</font>方法。分析出其中的关键一步是<font color = "rgb(226,238,250)">`je`</font>指令，最后还知道了这条指令在虚拟内存中的地址。

其实我们的目的非常简单，就是把<font color = "rgb(226,238,250)">`je`</font>指令换成<font color = "rgb(226,238,250)">`jne`</font>指令。到目前为止，只剩三步。

1. 算出<font color = "rgb(226,238,250)">`je`</font>指令的二进制码。
2. 算出<font color = "rgb(226,238,250)">`jne`</font>指令的二进制码。
3. 在二进制文件中，把算出<font color = "rgb(226,238,250)">`je`</font>指令的二进制码换成算出<font color = "rgb(226,238,250)">`jne`</font>指令的二进制码。

幸好，gdb调试器能够为我们做前两步。免去了我们完全不熟悉的从汇编到二进制码转换的过程。gdb调试器有一个<font color = "rgb(226,238,250)">`x/x`</font>命令，可以读取给定内存地址中的数据。

我们知道，程序运行的过程，简单来说其实就是二进制码从硬盘加载进内存，然后从程序入口开始运行的过程。我们不是汇编器，不善于做静态的、从汇编码到二进制码的转换工作。但是gdb调试器允许我们动态地、逆向的从内存中找到二进制码。

所以，距离成功还差最后一步！

### 二进制文件

所以，执行：

<font color = "rgb(226,238,250)">`x/x 0x00000001000cdfaf`</font>

可以得到如下的结果：

<font color = "rgb(226,238,250)">`0x1000cdfaf <_mh_execute_header+843695>:	0x83480774`</font>

这里的**0x83480774**就是16进制格式的程序二进制码。接下来就可以打开**Hex friend**软件对二进制码进行修改了。**Hex friend**把应用程序以16进制的形式展现出了，支持查找、替换功能。

按下<font color = "rgb(226,238,250)">`Command + F`</font>进行查找。

特别需要注意点是**字节序问题（Byte Order）**，Intel处理器一般是以**小端（Little endian）**进行存储，而在硬盘上的二进制码，则是以**大端（Big endian）**存储。所谓的**大端**，就是把数字的最高位放在最前面，**小端**则是把最高位放在最后面。

也就是说**0x83480774**作为一个**小端**数，它的**大端**形式应该是**74074883**,点击**Replace & Find**按钮之后，很不幸的事情出现了：这个数字不止出现了一次。

解决方案很简单，用同样的方法，看看下一条指令的的二进制码就可以了。执行：

<font color = "rgb(226,238,250)">`0x1000cdfb1 <_mh_execute_header+843695>:	0x83480774`</font>

得到：

<font color = "rgb(226,238,250)">`0x1000cdfaf <_mh_execute_header+843695>:	0x08c48348`</font>

用大端表示就是**4883c408**，这个数字的前四位和之前的数字的后四位刚好是相同的。这个不是巧合，因为不同的指令，二进制码长度不同。而gdb的<font color = "rgb(226,238,250)">`x/x`</font>指令总是读取相同长度的内存中的数据。

这一点并不影响破解Paw，但是如果想了解的非常透彻的话，可以用<font color = "rgb(226,238,250)">`otx`</font>命令查看：

![otx å‘½ä»¤](https://o8ouygf5v.qnssl.com/AppCrack/otx.png)

可以看到其实<font color = "rgb(226,238,250)">`eq`</font>指令的实际二进制码是**7407**。

现在终于确定了要被替换的数字式**74074883c408**，这里面包含了<font color = "rgb(226,238,250)">`eq`</font>指令的二进制码和接下来一些指令的二进制码。这些多余信息是为了唯一确定这组数的位置的。

**“7407”**由**“74”**和**“07”**两部分组成，查阅相关资料或者多找几个其他的<font color = "rgb(226,238,250)">`eq`</font>指令和<font color = "rgb(226,238,250)">`enq`</font>指令可以知道，<font color = "rgb(226,238,250)">`eq`</font>指令的二进制码是**“74”**而<font color = "rgb(226,238,250)">`eq`</font>指令的二进制码是**“75”**。

所以用来替换的数应该就是**“75074883c408”**。在**Hex friend**中填写好相关数据后选择**Replace**并保存。如图所示：

![æ–‡æœ¬æ›¿æ¢](https://o8ouygf5v.qnssl.com/AppCrack/replace.png)


至此，整个破解的过程就完成了。其实细想一下，我们只是**把一个4换成了5**而已！

### 文件签名

用修改过后的二进制文件替换原来文件后，打开程序总是会立刻报错。如果在命令行中运行，还可以看到**killed 9**的提示。

这是因为苹果为了保证软件的安全加入了**<font color=red>代码签名（CodeSignature）</font>**机制。在**Contents**文件夹下可以找到**_CodeSignature**文件夹和其中的**CodeResources**文件。任何对二进制文件的修改，都无法通过代码签名的检查。

关于代码签名的具体解释，和操作过程，可以看这篇文章：
> [《How to re-sign Apple's applications once they've been modified》](http://forums.macnn.com/79/developer-center/355720/how-re-sign-apples-applications-once/)

文章把每一步都描述得非常透彻，我就不重述了。按照文章所描述的，建立好自己的签名证书后，只要执行这条命令：


<font color = "rgb(226,238,250)">`codesign -f -s 证书名 /Applications/PawCrack.app/Contents/MacOS/Paw`</font>

其中证书名写自己创建的证书的名字，一切顺利的话，会得到这样的提示：
<font color = "rgb(226,238,250)">`/Applications/PawCrack.app/Contents/MacOS/Paw: replacing existing signature`</font>

代码重签名完成之后，就可以成功打开破解之后的App了。

打开App之后我们可以发现，**烦人**的Welcome页不见了。因为反转了判断逻辑，所以不执行<font color = "rgb(226,238,250)">`showWelcomeWindow`</font>方法了。

破解版的app可以在这里找到：[Paw破解版](http://download.csdn.net/detail/abc649395594/9291393)

## 总结

首先回顾一下整个破解过程。准备好工具之后，我们先从头文件里面搜索可以的方法名，再用反编译工具查看具体方法的汇编代码实现。结合基本的汇编语法和伪代码，了解整个方法的工作原理。最后修改if语句的逻辑从而完成破解。

其实由于大部分针对功能的限制，都是基于<font color = "rgb(226,238,250)">`if else`</font>语句进行判断的，也就是说对于相当多的软件，只要我们分析出它的逻辑，只需要把一个4改成5即可破解。

整个破解过程，除了巩固了操作系统的基础知识之外，我觉得对于iOS engineer来说还有一些其他的收获：

1. 严格遵守**“迪米特法则”**，把不必要对外提供的在.m文件里定义、实现。这样不仅防止被class-dump扫描到，也能减轻与你合作的同事开发时的负担。
2. 发布版本gcc编译时去掉<font color = "rgb(226,238,250)">`-g`</font>参数。我猜测，正是由于Paw这么做了，导致我无法用gdb调试器加断点。因为找不到函数的符号名。
3. 对于极为核心的部分，可以做适当的代码混淆。

做到以上几点非常轻松，但是足以防止数量广大，但又技术一般的tinkerer(比如作者本人)的捣鼓了。

## 参考资料

 1. [《Giving gdb permission to control other processes》](http://www.cnblogs.com/yishuiliunian/archive/2013/01/13/2858836.html)
 2. [《I Can Crack Your App With Just A Shell》](http://kswizz.com/2011-01-16/hacking-mac-apps/)
 3. [《How to re-sign Apple's applications once they've been modified》](http://forums.macnn.com/79/developer-center/355720/how-re-sign-apples-applications-once/)
 4. [Beginning Mac Hacking](http://www.mrspeaker.net/2011/01/06/mac-hacking/)