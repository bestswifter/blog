之前曾写过一篇文章:[如何大幅度提高 Mac 开发效率](https://bestswifter.com/efficient-mac/)，主要介绍了各种提高 Mac 效率的方法，美中不足的是忘记了介绍神器 Alfred。

更大的问题在于，不管是 Alfred 还是其他的应用，我们都是被动的使用者。也就是说我们只能学习如何使用现有的工具，却不能利用自己的编程技能去提高效率，而所谓的工具，唯一的目的和价值就是提高自己的工作效率，解决痛点，而非被人学习如何使用。

这篇文章的主要目的就是介绍如何在 Alfred 打造自己的 workflow，从而能自己发现、并解决那些可以被自动化脚本取代的、无聊的工作。作为示范，我提供了两个插件:

1. 快速打开&关闭 shadowsocks 连接
2. 截图后快速上传到七牛

他们使用不同的技术制作而成，但宗旨只有一个:**“找到一切可以被自动化的流程并且用脚本来完成”**，都开源在[我的Github](https://github.com/bestswifter/my-workflow)。

# Workflow

workflow 是 Alfred 中最强大的部分，它能把常见工作转变成一个工作流

# AppleScript

如果你使用 GoAgentX 这个软件，并且用 shadowsocks 翻墙，那么频繁的关闭和打开连接应该是无法避免的。

为了避免频繁的移动鼠标和点击，我希望用命令的方式来打开和关闭 shadowsocks 连接，如下图所示:

![演示](https://o8ouygf5v.qnssl.com/alfred/GoAgentX-Alfred.gif )

这其实类似于按键精灵的思想，因为我们希望有人能自动点击那个连接按钮或者关闭按钮。这种情况下可以考虑使用 AppleScript。它的作用是方便我们和程序进行交流，从而执行一系列程序内置的操作指令。

这里我不想自己介绍它的语法，因为实在是太白话了，文末的参考资料中也有详细的解释。来看一下第一个插件的脚本:

```applescript
on alfred_script(q)
  -- 根据自己的需要修改profileName
  set profileName to "shadowsocks"
  tell application "GoAgentX"
    if q is "c" then
		toggle status of profileName to running
	else if q is "d" then
		toggle status of profileName to stopped
	end if
  end tell
end alfred_script
```

如果你略具备编程经验，你一定会好奇这行代码是怎么写出来的，以及他们的语法是什么:

```applescript
toggle status of profileName to running
```

首先打开系统应用:脚本编辑器(Script)，然后选择 文件->打开字典，找到 GoAgentX 这个应用，你就会发现所有的语法都在这里了:

![GoAgentX支持的所有脚本命令](https://o8ouygf5v.qnssl.com/1468480517.png )

所以，有心的开发者日后在自己的应用中也可以考虑添加对 AppleScript 的支持。

# Python

如果是和外部程序交互，比如网络请求，文件读写，执行 bash 脚本等等，就不再适合使用 AppleScript 这么简单的脚本了，推荐使用 Python 进行编程。

长久以来，写博客时把图片上传到图床一直是一个繁琐的任务:

1. 截图
2. 打开截图保存目录，复制图片
3. 去七牛上传
4. 复制 url
5. 粘贴到 markdown 文本中

通过 workflow，我们可以把上述步骤简化为简单的三步，省略大量时间:

1. Command+Ctrl+A 截图(图片不要保存在本地，存在剪贴板中即可)
2. 在 Alfred 中输入 `gn` 
3. 图片会自动上传，并且把 Url 复制到剪贴板中，你只要按下 Command + V 即可

具体的实现也不复杂，由于已经开源所以就不分析了，简单介绍一下用法:

1. 首先运行在 Alfred 中输入 `qnconf`，后面要加上 AK 和 SK，这是你的密钥，可以从官网获取，输入如下:
    
       qnconf xxx xxx

2. 在 workflow 插件的文件夹中，有一个 `conf.txt` 文件，你需要打开它，设置上传到哪个 bucket，以及你的图床前缀，可以根据我现有的文件做修改。
3. 使用时，确保剪贴板中有图片，在 Alfred 中输入 `qn` 后面也可以指定一个参数名，表示上传图片的名称。如果不写则默认是当前时间戳。
4. 运行后，图片的 url 已经复制在你的剪贴板中了，尽情使用吧。

# 写在最后

这篇文章的目的不是介绍 workflow 的开发，也不是介绍 Python 的语法，这些资料网上应有尽有。

**学习一门脚本语言，结合一个伟大的应用，大幅度提升自己的效率**，这种一举两得的事情实在是再美好不过了。

# 参考资料

1. [AppleScript的终极入门手册](http://www.jayz.me/?p=267)
2. [Alfred workflow 开发指南](http://myg0u.com/python/2015/05/23/tutorial-alfred-workflow.html)
3. [pngpaste](https://github.com/jcsalterego/pngpaste)