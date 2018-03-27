## 背景

网上有很多使用Storyboard完成`UIScrollview`的例子，但是纯代码的例子却不多。有限的一些例子大多也是外国开发者用VFL写的。而这篇文章基于swift语言和SnapKit分析了如何用纯代码加Autolayout写`UIScrollview`，完整代码已经上传到我的[github](https://github.com/bestswifter/MySampleCode/tree/master/AutolayoutScrollViewInCode)。

在正文中，我会分析其中的关键代码。对于Autolayout，**绝对不可取**的态度是不停的试几个约束，一旦发现好用，也不管其原理，就放手不管了。事实上，我们写的每一个约束，都要明白它存在的价值是什么，要做到不写一个无用的约束，不漏一个必要的约束，明白为什么某种写法有效，而另一种写法就无效。

废话不多说，估计大家用`UIScrollView`时，都有过被Autolayout坑的经历，要么是布局不对，要么不能滑动，以及其他匪夷所思的bug。这与Autolayout和`UIScrollView`各自的特性有关。

# 理论分析

首先，我们知道Autolayout改变了传统的以frame为主的布局思想。它其实是一种相对布局，核心思想是视图与视图之间的位置关系。比如，我们可以根据矩形的起始横坐标、纵坐标、长和宽这四个变量确定它的位置。或者，如果已经确定矩形A的位置，只要知道矩形B每条边的和A对应边之间的距离，也能确定B的位置。前者就是frame的思想，它基于绝对数值，而后者是Autolayout的思想，它基于偏移量的概念。

其次，`UIScrollView`有自己的frame也就是我们在屏幕上能看到的区域。它还有一个`contentSize`的概念。在使用frame布局的时候，我们一般先设置好子视图的位置，最后再设置`contentSize`,它会将所有的子视图包含在内。于是通过滑动，我们就可以在有限的布局中，看到所有的内容了。

但是在Autolayout时代，为了简化布局，我们希望`contentSize`能够自动设置。比如有一个scrollView，它有两个子视图。frame分别为(x: 0, y: 0, width: 10, height: 10)和(x: 10, y: 0, width: 10, height: 10)，那么我们自然会认为这两个视图左右并排排列，`contentSize`为(x: 0, y: 0, width: 20, height: 10)：

![自动计算contentSize](http://images.bestswifter.com/Autolayout+Scrollview/sample1.png)

这种把若干个子视图合并，得出`contentSize`的能力，人类是天生具备的，但是计算机却不是这样。仅凭以上信息，程序无法推断出真正的`contentSize`。原因在于，我们没有明确的告诉系统，在这两个子视图拼接而成的区域以外，还有没有区域应该被`contentSize`包含。

也就是说，`contentSize`也有可能是下图中的阴影部分：

![更大的contentSize](http://images.bestswifter.com/Autolayout+Scrollview/sample2.png)

如果需要指定`contentSize`就是两个正方形拼接而成的区域，我们还需要提供四个信息：

1. 左边的正方形的左侧的边，距离contentSize左边的距离为0
2. 右边的正方形的右侧的边，距离contentSize右边的距离为0

……

通过以上的分析，我们可以看到，其实`contentSize`是依赖于子视图自身的大小，和上下左右四个方向的留白大小计算出的。而`UIScrollView`的leading/trailing/top/bottom是相对于它的`contentSize`而不是`bounds`来确定的。所以如果你写这样的代码，布局是肯定不会生效的：

```swift
subview.snp_makeConstraints { (make) -> Void in
    make.edges.equalTo(scrollView).offset(5)
}
```

因为我们其实是在根据`UIScrollView`的leading/trailing/top/bottom来确定子视图的位置，而我们已经分析过，`UIScrollView`的leading/trailing/top/bottom是相对于自己的`contentSize`而言的。而`contentSize`又是根据子视图位置决定的。这就变成了一种你依赖我，我又依赖你的情况。

为了打破这种循环依赖，为子视图添加约束的**两个要求**是：

1. 它不依赖于任何与scrollview有关布局，也就是不能参考scrollview的位置和大小。
2. 它不仅要确定过自己的大小，还要确定自己与contentSize四周的距离。


第二个要求意思是说，正常使用autolayout时，我们确定一个矩形在水平方向上的范围，只要知道它的左边距离它左边的矩形有多远，以及它有多宽即可。但是在`UIScrollView `中布局时，还需要告诉`UIScrollView `，它的右边距离右边的视图有多远。这样`contentSize`才能确定。否则`UIScrollView `就不知道`contentSize`向右可以延伸多少。在竖直方向上也是同理。

**这两大要求一定要牢记！**接下来我们的代码都将围绕如何满足这两大要求展开。

# 动手实践

明白了问题的理论背景后，我们通过一个具体的需求，来看看正确的代码怎么写，以下面这个效果为例：

![任务目标](http://images.bestswifter.com/Autolayout+Scrollview/1.pic.png)

如图所示，中间是一个`UIScrollView`,它的背景颜色是黄色。红色部分我们称之为`box`，它是一个普通的，红色背景的`UIView`。也就是说我们向`UIScrollView`中添加了多个`box`，每个子`box`之间间隔一定距离。我们分步实现这个功能

## 使用container

首先我们介绍一种使用Container的方法。

### 第一步：为scrollView添加约束


```swift
let scrollView = UIScrollView()
view.addSubview(scrollView)
scrollView.snp_makeConstraints { (make) -> Void in
    make.centerY.equalTo(view.snp_centerY)
    make.left.right.equalTo(view)
    make.height.equalTo(topScrollHeight)
}
```

我们之前说过，使用Autolayout时，不用考虑frame布局。所以直接创建一个`scrollView `对象。需要先把`scrollView `添加到父视图上才能添加约束。

对`scrollView `添加约束没有什么难点，就像我们给其他视图添加约束一样。这里表示`scrollView`和父视图左右对齐，居中显示。

### 第二步：为container添加约束

```swift
scrollView.addSubview(containerView)
containerView.snp_makeConstraints { (make) -> Void in
    make.edges.equalTo(scrollView)
    make.height.equalTo(topScrollHeight)
}
```

这里对`container`的约束非常重要，第一个约束表示自己上、下、左、右和`contentSize`的距离为0，因此只要`container`的大小确定，`contentSize`也就可以确定了，因为此时它和`container `大小、位置完全相同。

第二个约束直接通过一个数值，确定`container `的高度。避免了依赖`scrollview `布局。这样一来，`scrollview `就变成水平的了。`container `的宽度直接决定了`scrollview `的宽度。

### 第三步：添加box

```swift
for i in 0...5 {
    let box = UIView()
    containerView.addSubview(box)
    
    box.snp_makeConstraints(closure: { (make) -> Void in
        make.top.height.equalTo(containerView)  // 确定top和height之后，box在竖直方向上完全确定
        make.width.equalTo(boxWidth)		//确定width后，只要再确定left，就可以在水平方向上完全确定
        if i == 0 {
            make.left.equalTo(containerView).offset(boxGap / 2)  //第一个box的left单独处理
        }
        else if let previousBox = containerView.subviews[i - 1] as? UIView{
            make.left.equalTo(previousBox.snp_right).offset(boxGap)  // 在前一个box右侧15个距离
        }
        if i == 5 {
            containerView.snp_makeConstraints(closure: { (make) -> Void in
                make.right.equalTo(box)  // 确定container的右侧边界。
            })
        }
    })
}
```

对`box`的约束看似复杂，其实非常简单。因为`scrollview `在Autolayout下的布局，难点就在于子视图布局时约束比较多。但现在，我们通过一个`container`已经隔离了，也就说我们又回归了常规的Autolayout布局。以水平方向为例，我们只要确定`left`和`width`即可。

在最后一个`if`语句中，我们为`container `添加了右侧的约束。这样就确定了`container`的宽度。由于`container`封装了所有的`box`，所以对于`scrollview `来说，它的子视图只有一个，就是`container`，而`container`自身的大小，上下左右四个方向和`contentSize`距离在之前的约束中已经被定义为0，`contentSize`也就可以确定了。

## 使用外部视图

除了使用`container `以外，我们还可以使用外部的视图确定子视图的位置。这种方法，步骤较少，和之前一样，第一步是创建`scrollView`并添加约束。接下来我们直接添加子视图：

```swift
box.snp_makeConstraints(closure: { (make) -> Void in
    make.top.equalTo(0)
    make.bottom.equalTo(view).offset(-(ScreenHeight - topScrollHeight) / 2)  // This bottom can be incorret when device is rotated
    make.height.equalTo(topScrollHeight)

    make.width.equalTo(boxWidth)
    if i == 0 {
        make.left.equalTo(boxGap / 2)
    }
    else if let previousBox = scrollView.subviews[i - 1] as? UIView{
        make.left.equalTo(previousBox.snp_right).offset(boxGap)
    }

    if i == 5 {
        make.right.equalTo(scrollView)
    }
})
```

这时候，`box`是直接add到`scrollView`上的。我们直接指定它的`top`为0。前三个约束分别指定了`box`的顶部、底部和高度。这样就在竖直方向上满足了两大要求中的第二个要求。对于`bottom`的约束，它的参考物是`view`，这就是所谓的外部视图。

接下来我们分别为`width`和`left`添加了约束。而且只要对最后一个`box`添加`right`约束即可在水平方向上满足第二个要求。由于我们的布局依赖于外部的视图，所以自然满足第一个要求，因此这种写法也是可以的。

## Container与外部视图的优缺点

与`container`相比，使用外部视图除了代码量可能略少以外，我实在想不到它还有什么优点。

首先，一旦我们使用了`container`，首先它天然满足第一个要求，因为它并没有进行布局，只是让`contentSize`与自己等大，然后设置自己的大小。而且它几乎已经满足了第二个要求。只要我们最后确定它的宽度或高度即可。其次，在`container`内部，子视图布局不用考虑满足第二个要求，因为`container`已经隔离了这一切，我们要做的只是按照习惯，确定子视图的位置，这样`container`的位置也会随着子视图确定。

其次，我发现的使用外部视图布局的缺点就至少有三个：

1. 它依赖外部视图进行定位，这样的写法不够优雅
2. 观察代码中对于bottom属性的约束，它不能完美适配旋转屏幕后的视图。因为此时的屏幕长和宽会对调。而且目测没有什么好的解决方案。
3. 布局过程中容易踩到坑，比如对于`left`属性的约束，如果你的代码是这样的：

```swift
make.left.equalTo(view).offset(boxGap / 2)
```

它和原来的写法几乎是等价的。但你仔细分析，或者试着滑动`scrollView`时，一定会大吃一惊。如果你不能一眼看出来这种写法的问题所在，那我建议你运行代码体验一下，并且以后尽量避免这种写法。

> 最后重复一下，代码地址在[https://github.com/bestswifter/MySampleCode/tree/master/AutolayoutScrollViewInCode](https://github.com/bestswifter/MySampleCode/tree/master/AutolayoutScrollViewInCode)，可以下载下来把玩研究一番，如果觉得对你有帮助，请给一个star。