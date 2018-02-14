在使用UIKit的过程中，性能优化是永恒的话题。很多人都看过分析优化滑动性能的文章，但其中不少文章只介绍了优化方法却对背后的原理避而不谈，或者是晦涩难懂而且读者缺乏实践体验的机会。不妨思考一下下面的问题自己是否有一个清晰的认识：

1. 为什么要把控件尽量设置成不透明的，如果是透明的会有什么影响，如何检测这种影响？
2. 为什么cell中的图片，尽可能要使用正确的大小、格式，如果错误会有什么影响，如何检测这种影响？
3. 为什么设置阴影和圆角有可能影响滑动时流畅度？
4. `shouldRasterize`和离屏渲染的关系是什么，何时应该使用？

本文会结合Instrument分析影响性能的因素，提出优化方案并解释背后的原理，项目初始demo的下载地址在[我的Github](https://github.com/bestswifter/MySampleCode/tree/master/GraphicsPerformance-Starter)，强烈建议每一位读者下载下来随着我一步一步调试、优化。如果觉得对自己有帮助，可以给一个Star表示支持。后面的图片较多，流量党慎入。

# 基本概念

打开项目后，只要CustomTableCell.swift文件即可，它实现了自定义的`UITableViewCell`以及内部的UI布局，因为重点在于性能优化，代码实现的就比较随意。

首先按下`Command + I`打开Instrument，本文主要用到的是Core Animation工具：

![打开Core Animation调试](http://images.bestswifter.com/UIKitPerformance/Instrument.png)

注意这个调试必须使用真机，点击左上角的红色圆圈就会开始录制。新手可能不太熟悉，这里简单介绍一下调试界面：

![调试界面](http://images.bestswifter.com/UIKitPerformance/Introduce.png)

我们需要了解两个两个区域：

1. 这里记录了实时的fps数值，有些地方是0是因为屏幕没有滑动
2. 这是重中之重，接下来我会带大家逐个理解、体验这些调试选项

有过游戏经验的人也许对**fps**这个概念比较熟悉。我们知道任何屏幕总是有一个刷新率，比如iphone推荐的刷新率是60Hz，也就是说GPU每秒钟刷新屏幕60次，因此两次刷新之间的间隔为16.67ms。这段时间内屏幕内容保持不变，称为**一帧(frame)**，fps表示**frames per second**，也就是每秒钟显示多少帧画面。对于静止不变的内容，我们不需要考虑它的刷新率，但在执行动画或滑动时，fps的值直接反映出滑动的流畅程度。

# 调试、优化

### 图层混合

首先我们要明白像素的概念，屏幕上每一个点都是一个像素，像素有R、G、B三种颜色构成(有时候还带有alpha值)。如果某一块区域上覆盖了多个layer,最后的显示效果受到这些layer的共同影响。举个例子，上层是蓝色(RGB=0,0,1),透明度为50%，下层是红色(RGB=1,0,0)。那么最终的显示效果是紫色(RGB=0.5,0,0.5)。这种颜色的混合(blending)需要消耗一定的GPU资源，因为实际上可能不止只有两层。如果只想显示最上层的蓝色，可以把它的透明度设置为100%，这样GPU会忽略下面所有的layer，从而节约了很多不必要的运算。

第一个调试选项"Color Blended Layers"正是用于检测哪里发生了图层混合，并用红色标记出来。因此我们需要尽可能减少看到的红色区域。一旦发现应该想法设法消除它。开始调试后勾选这个选项，我们在手机上可以看到如下的场景：

![Color Blended Layers](http://images.bestswifter.com/UIKitPerformance/blendedlayer.png)

很多文章里说把控件设置为`opaque = true`，其原理就是希望避免图层混合，然而这种调优一般情况下用处不大。因为`UIView`的`opaque`属性默认值就是`true`，也就是说只要不是人为设置成透明，都不会出现图层混合。比如demo中就没有任何透明的控件。

对于`UIImageView`来说，不仅它自身需要是不透明的，它的图片也不能含有alpha通道，这就是为什么图中第三个图片是绿色，而前两个图片是红色的原因。由于本人对PS和图像几乎一窍不通，恕我不能演示如何消除这些图片的红色。我从网上找了一个美女的头像来说明，图像自身的性质可能会对结果有影响，因此如果你确定自己的代码没有问题，而且出现了图层混合，请联系美工或后台解决。

个人认为比`opaque`属性更重要的是`backgroundColor`属性，如果不设置这个属性，控件依然被认为是透明的，所以我们做的第一个优化是在`CustomTableCell`类的`init`方法中添加一行代码：

```
label.backgroundColor = UIColor.whiteColor()
```

虽然在白色背景下，这行代码无法肉眼看到效果，但重新调试后我们可以发现label的红色消失了。也正是因为对背景颜色的不重视，它成了影响滑动性能的第一个杀手。

PS：如果label文字有中文，依然会出现图层混合，这是因为此时label多了一个`sublayer`，如果有好的解决办法欢迎告诉我。

### 光栅化

光栅化是将一个layer预先渲染成位图(bitmap)，然后加入缓存中。如果对于阴影效果这样比较消耗资源的静态内容进行缓存，可以得到一定幅度的性能提升。demo中的这一行代码表示将label的layer光栅化：

```swift
label.layer.shouldRasterize = true
```

Instrument中，第二个调试选项是“Color Hits Green and Misses Red”，它表示如果命中缓存则显示为绿色，否则显示为红色，显然绿色越多越好，红色越少越好。勾选这个选项后我们看到如下的场景：

![Color Hits Green and Misses Red](http://images.bestswifter.com/UIKitPerformance/rasterize.png)

光栅化的核心在于缓存的思想。我们自己动手把玩一下，可以发现以下几个有意思的现象：

1. 上下微小幅度滑动时，一直是绿色
2. 上下较大幅度滑动，新出现的label一开始是红色，随后变成绿色
3. 如果静止一秒钟，刚开始滑动时会变红。

这是因为layer进行光栅化后渲染成位图放在缓存中。当屏幕出现滑动时，我们直接从缓存中读取而不必渲染，所以会看到绿色。当新的label出现时，缓存中没有个这个label的位图，所以会变成红色。第三点比较关键，缓存中的对象有效期只有100ms，即如果在0.1s内没有被使用就会自动从缓存中清理出去。这就是为什么停留一会儿再滑动就会看到红色。

光栅化的缓存机制是一把双刃剑，先写入缓存再读取有可能消耗较多的时间。因此光栅化仅适用于较复杂的、静态的效果。通过Instrument的调试发现，这里使用光栅化经常出现未命中缓存的情况，如果没有特殊需要则可以关闭光栅化，所以我们做的第二个优化是注释掉下面这行代码：

```swift
//	label.layer.shouldRasterize = true
```

光栅化会导致离屏渲染，这一点待会儿会讲。

### 颜色格式

像素在内存中的布局和它在磁盘中的存储方式并不相同。考虑一种简单的情况：每个像素有R、G、B和alpha四个值，每个值占用1字节，因此每个像素占用4字节的内存空间。一张1920*1080的照片(iPhone6 Plus的分辨率)一共有2,073,600个像素，因此占用了超过8Mb的内存。但是一张同样分辨率的PNG格式或JPEG格式的图片一般情况下不会有这么大。这是因为JPEG将像素数据进行了一种非常复杂且可逆的转化。

当我们打开JPEG格式的图片时，CPU会进行一系列运算，将JPEG图片解压成像素数据。显然这个工作会消耗不少时间，所以不应该在滑动时进行，我们应该预先处理好图片。借用WWDC上的一页PPT来说明：

![显示流程](http://images.bestswifter.com/UIKitPerformance/pipeline.png)

Commit Transaction和Decode在同一帧内进行，如果这两个操作的耗时超过16.67ms，Draw Calls就会延迟到下一帧，从而导致fps值的降低。下面是Commit Transaction的详细流程：

![解码与转换](http://images.bestswifter.com/UIKitPerformance/commit.png)

在第三步的Prepare中，CPU主要处理两件事：

1. 把图片从PNG或JPEG等格式中解压出来，得到像素数据
2. 如果GPU不支持这种颜色各式，CPU需要进行格式转换

比如应用中有一些从网络下载的图片，而GPU恰好不支持这个格式，这就需要CPU预先进行格式转化。第三个选项“Color Copied Images”就用来检测这种实时的格式转化，如果有则会将图片标记为蓝色。

遗憾的是由于我对图片格式不太了解，也不会使用相关工具，并没有能模拟出触发这个选项的场景。我们要记住的是，如果调试时发现有图片被标记为蓝色，说明图片格式出现了一些问题。

### 图片大小

第四个选项的使用场景不多，我们直接看一下第五个选项“Color Misaligned Images”。它表示如果图片需要缩放则标记为黄色，如果没有像素对齐则标记为紫色。勾选上这个选项并进行调试，可以看到如下场景：

![图片缩放](http://images.bestswifter.com/UIKitPerformance/scale.png)

在demo中，每个`UIImageView`的大小都是180x180，而只有第二张图片的像素大小是360x360。因此除了第二张图片，其他的图片都需要被缩放。图片的缩放需要占用时间，因此我们要尽可能保证无论是本地图片还是从网络或取得图片的大小，都与其frame保持一致。

第三个优化是调整所有图片的像素大小以避免不必要的缩放。

### 离屏渲染

离屏渲染表示渲染发生在屏幕之外，你可能认为这是一句废话。为了真正解释清楚什么是离屏渲染，我们先来看一下正常的渲染通道(Render-Pass)：

![正常渲染通道](http://images.bestswifter.com/UIKitPerformance/renderpass.png)

首先，OpenGL提交一个命令到Command Buffer，随后GPU开始渲染，渲染结果放到Render Buffer中，这是正常的渲染流程。但是有一些复杂的效果无法直接渲染出结果，它需要分步渲染最后再组合起来，比如添加一个蒙版(mask)：

![离屏渲染](http://images.bestswifter.com/UIKitPerformance/offscreenpass.png)

在前两个渲染通道中，GPU分别得到了纹理(texture，也就是那个相机图标)和layer(蓝色的蒙版)的渲染结果。但这两个渲染结果没有直接放入Render Buffer中，也就表示这是离屏渲染。直到第三个渲染通道，才把两者组合起来放入Render Buffer中。离屏渲染意味着把渲染结果临时保存，等用到时再取出，因此相对于普通渲染更占用资源。

第六个选项“Color Offscreen-Rendered Yellow”会把需要离屏渲染的地方标记为黄色，大部分情况下我们需要尽可能避免黄色的出现。离屏渲染可能会自动触发，也可以手动触发。以下情况可能会导致触发离屏渲染：

> 1. 重写drawRect方法
> 2. 有mask或者是阴影(layer.masksToBounds, layer.shadow*)，模糊效果也是一种mask
> 3. layer.shouldRasterize = true

前两者会自动触发离屏渲染，第三种方法是手动开启离屏渲染。

开始调试并勾选“Color Offscreen-Rendered Yellow”，会看到这样的场景：

![离屏渲染](http://images.bestswifter.com/UIKitPerformance/offscreenrender.png)

如果没有进行第二步优化，你会发现label也是黄色。可以看到tabbar和statusBar也是黄色，这是因为它们使用了模糊效果。图片也是黄色，这说明它也进行了离屏渲染，观察源码后发现主要原因是它使用了阴影，接下来我们进行第四个优化，在设置阴影效果的四行代码下面添加一行：

```swift
imgView.layer.shadowPath = UIBezierPath(rect: imgView.bounds).CGPath
```

这行代码制定了阴影路径，如果没有手动指定，Core Animation会去自动计算，这就会触发离屏渲染。如果人为指定了阴影路径，就可以免去计算，从而避免产生离屏渲染。

设置`cornerRadius`本身并不会导致离屏渲染，但很多时候它还需要配合`layer.masksToBounds = true`使用。根据之前的总结，设置`masksToBounds`会导致离屏渲染。解决方案是尽可能在滑动时避免设置圆角，如果必须设置圆角，可以使用光栅化技术将圆角缓存起来：

```swift
// 设置圆角
label.layer.masksToBounds = true
label.layer.cornerRadius = 8
label.layer.shouldRasterize = true
label.layer.rasterizationScale = layer.contentsScale
```

### 快速路径

还记得之前将离屏渲染和渲染路径时的示意图么，离屏渲染的最后一步是把此前的多个路径组合起来。如果这个组合过程能由CPU完成，就会大量减少GPU的工作。这种技术在绘制地图中可能用到。

第七个选项“Color Compositing Fast-Path Blue”用于标记由硬件绘制的路径，蓝色越多越好。

### 变化区域

刷新视图时，我们应该把需要重绘的区域尽可能缩小。对于未发生变化的内容则不应该重绘，第八个选项“Flash updated Regions”用于标记发生重绘的区域。一个典型的例子是系统的时钟应用，绝大多数时候只有显示秒针的区域需要重绘：

![重绘区域](http://images.bestswifter.com/UIKitPerformance/flash.png)

# 总结

如果你一步一步做到了这里，我想一定会有不少收益。不过，学而不思则罔，思而不学则殆。动手实践后还是应该总结提炼，优化滑动性能主要涉及三个方面：

### 避免图层混合

1. 确保控件的`opaque`属性设置为`true`，确保`backgroundColor`和父视图颜色一致且不透明
2. 如无特殊需要，不要设置低于1的`alpha`值
3. 确保`UIImage`没有alpha通道

### 避免临时转换

1. 确保图片大小和`frame`一致，不要在滑动时缩放图片
2. 确保图片颜色格式被GPU支持，避免劳烦CPU转换

### 慎用离屏渲染

1. 绝大多数时候离屏渲染会影响性能
2. 重写`drawRect`方法，设置圆角、阴影、模糊效果，光栅化都会导致离屏渲染
3. 设置阴影效果是加上阴影路径
4. 滑动时若需要圆角效果，开启光栅化

### 实战

本文的demo可以在[我的Github](https://github.com/bestswifter/MySampleCode/tree/master/GraphicsPerformance-Starter)上下载，然后一步一步自己体验优化过程。但demo毕竟是刻意搭建的一个环境，我会在我自己的仿写的[简书app](https://github.com/WL201314/MJianshu)上不断进行实战优化，欢迎共同学习交流。

# 参考资料：

1. [绘制像素到屏幕上](http://objccn.io/issue-3-1/)，原文：[Getting Pixels onto the Screen](https://www.objc.io/issues/3-views/)
2. [Advanced Graphics and Animations for iOS Apps](https://developer.apple.com/videos/play/wwdc2014-419/)：这是2014年WWDC Session 419，强烈建议看一遍。
3. [如何正确地写好一个界面](http://oncenote.com/2015/12/08/How-to-build-UI/)
4. [Mastering UIKit Performance](https://yalantis.com/blog/mastering-uikit-performance/)

还有一些高质量的问答：

1. [What triggers “Color Copied Images” and “Color Hits Green and Misses Red” in Instruments?](http://stackoverflow.com/questions/6302029/what-triggers-color-copied-images-and-color-hits-green-and-misses-red-in-ins)
2. [UILabel is marked as red when `Color Blended Layers` is selected](http://stackoverflow.com/questions/34895641/uilabel-is-marked-as-red-when-color-blended-layers-is-selected)
3. [What triggers offscreen rendering, blending and layoutSubviews in iOS?](http://stackoverflow.com/questions/13158796/what-triggers-offscreen-rendering-blending-and-layoutsubviews-in-ios)