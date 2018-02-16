自己做了一个模仿简书的小项目练手，主要布局是上面的scrollview有一排label，下面的scrollview有多个UITableView。点击上面的label，下面就可以显示不同的页面。具体效果可以打开简书官方的APP查看，很多新闻软件也是这种效果。

一开始的思路就是加载所有ViewController，因为是TableView，所以每个TableView还有自己的DataSource，真机运行了一下，发现占用内存大概是36M左右。于是我开始着手对这种原始的实现方案进行逐步优化，主要是内存占用相关的，以及一些其他的小技巧。

项目在Github开源，本文涉及到的相关代码都可以自行查看。项目地址：[MJianshu](https://github.com/Wl201314/MJianshu)

![优化前内存](https://user-gold-cdn.xitu.io/2018/2/16/1619ef2841dad7df?w=600&h=587&f=jpeg&s=36182)

# 优化一：分离DataSource

为了轻量化`UIViewController`，同时也为了后期的解耦，我首先把`DataSource`从`UIViewController`中分离出来。思路是在`UIViewController`中引用一个`DataSource`对象，然后把`table`的dataSource属性设置成这个变量而不是自己，用代码描述就是：

```swift
// UIViewController.swift
var dataSource = ContentTableDatasource()

tableView.dataSource = dataSource
```

把DataSource相关的代理方法都放到`ContentTableDatasource`中去：

```swift
extension ContentTableDatasource {
    func tableView(tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        //行数
    }

    func tableView(tableView: UITableView, cellForRowAtIndexPath indexPath: NSIndexPath) -> UITableViewCell {
        //返回cell
    }
}
```

这样做的好处在于，`UIViewController`对具体的数据获取一无所知，它只负责给`table`委派数据源的任务。只要改变数据源，`table`的内容就可以改变。这也符合MVC模式中M和C的解耦。更详细的介绍在objc.io的[Lighter View Controllers](https://www.objc.io/issues/1-view-controllers/lighter-view-controllers/)一文中。

# 优化二：重用ViewController

如果不考虑点击顶部标签的情况，也就是只能滑动`BottomScrollview`，我们可以注意到一个事实。比如当前我在第五页，不管我要滑到其他的任何一页，都必须经过第四页或第六页。也就是说在这种情况下，除了4、5、6这三页的`UIViewController`，其他的都是无用的。一旦我向左滑到第四页，那么第六页的`UIViewController`也是无用的，它可以被重复利用，装载第三页所显示的`UIView`

所以，思路就是模仿`UITableView`的重用机制维护一个队列，实现`UIViewController`的重用。每当一个`UIViewController`变成无用的，就放入重用队列。需要`UIViewController`时先从重用队列中找，如果找不到就新建。这样一来内存中最多只会保存三个`UIViewController`的实例，所以占用内存大幅度降低。核心代码如下：

```swift
func scrollViewDidScroll(scrollView: UIScrollView) {
    // 加载即将出现的页面
    loadPage(page)
}

func loadPage(page: Int) {
    guard currentPage != page else { return }  //还在当前页面就不用加载了
    currentPage = page

    var pagesToLoad = [page - 1, page, page + 1]  // 筛选出需要加载的页面，一般只有一个
    var vcsToEnqueue: Array<ContentTableController> = []  // 把用不到的ViewController入队
}

func addViewControllerForPage(page: Int) {
    let vc = dequeueReusableViewController()  // 从队列中获取VC
    vc.pageID = page
    // 添加视图
}

func dequeueReusableViewController() -> ContentTableController {
    if reusableViewControllers.count > 0 {
        return reusableViewControllers.removeFirst() // 如果有可以重用的VC就直接返回
    }
    else { //否则就创建。程序刚开始运行的时候一般需要执行这一步
        let vc = ContentTableController()
        return vc
    }
}
```

关于重用队列，可以参考这个项目：[Reuse](https://github.com/allenhsu/UIScrollView-Samples/tree/master/Reuse)

# 优化三：点击Label后的过渡

如果从第一页滑动到第三页，那么第二页也会快速闪过。这样会导致用户体验比较差。我的思路是首先在第二页的位置上覆盖一个和第一页一模一样的`UIView`，然后不加动画的切换到第二页。这一瞬间用户感觉不到任何变化。然后再有动画的滑动到第三页。滑动完成之后需要移除这个临时添加的`UIView`，关键步骤如下所示

```swift
var maskView = UIView()
maskView = bottomScrollViewController.currentDisplayViewController()?.view // 获取用于遮盖的view

bottomScrollView.addBottomViewAtIndex(targetPage - 1, view: maskView) // 把view添加到目标页的前一页
buttomScrollView.bottomScroll.setContentOffset(CGPointMake(previousOffSetX, 0), animated: false)  //无动画滑动
buttomScrollView.bottomScroll.setContentOffset(CGPointMake(offSetX, 0), animated: true) //有动画滑动

func scrollViewDidEndScrollingAnimation(scrollView: UIScrollView) {
    maskView.removeFromSuperview()  // 滑动结束后移除临时视图
}
```

实际操作远比这个复杂。因为要实现`UIViewController`的重用，所以在`scrollViewDidScroll`这个代理方法中需要时刻监听滑动状态并加载下一页。在点击Label的时候需要禁掉这个特性。

总的来说，点击Label的切换和滑动切换页面并不是同一个原理，所以要保证他们之间的逻辑互不干扰

# 优化四：缓存DataSource

最初的逻辑是每个`UIViewController`自己处理自己的`dataSource`，现在因为在`BottomScrollview`中处理`UIViewController`的重用逻辑，所以dataSource的缓存和获取也就一并放在这里处理了。每个`UIViewController`重用时都会根据自己的页数去缓存中查找`dataSource`是否已经存在，如果已经存在的话就直接获取了。关键代码如下所示：

```swift
var dataSources: [Int: ContentTableDatasource] = [:]  // 键是页数，值是datasource对象

func bindDataSourceWithViewController(viewController: ContentTableController, page: Int) {
    if dataSources[page] == nil {  // 如果不存在，就去新建datasource
        dataSources[page] = ContentTableDatasource(page: page)
    }
    viewController.dataSource = dataSources[page]
}
```

实际上`dataSource`也可以重用，但是这样做并不能节省太多内存，反而会导致`dataSource`中内容的反复切换，有点得不偿失

# 防掉坑指南

最后再谈一谈`UIScrollView`中的一些坑，之前也写过一篇文章——[史上最简单的UIScrollView+Autolayout出坑指南](http://www.jianshu.com/p/f7f1ba67c3ca)，主要是关于`UIScrollView`在Autolayout下的布局问题。在后续的开发过程中，还是遇到了一些值得注意的地方。

因为`UIScrollView`是可以滑动的，所以对它的布局约束要格外小心。举个例子，一个子视图的`left`已经确定，这时候不管设置它的`right`约束还是`width`约束都可以固定它的位置。但是在`UIScrollView`，千万不要设置`right`约束。否则你可以想象一下，有一个橡皮筋，一端被固定，另一端被拉伸的感觉：

```swift
make.right.equalTo(view) // 滑动时视图会被拉伸
make.width.equalTo(viewWidth) // 正确
```

这样的bug非常难找到，所以我个人的经验是，在对`UIScrollView`的子视图布局时，尽量不要用两端的位置来确定视图自己的长度，而是应该通过自己长度确定另一端的位置。或者，干脆不要依赖于外部视图布局，而是用一个`Container`容器。这也是我在之前的文章中强烈推荐的方法。

# 成果：

内存占用显著减少，只有大约原来的一半。考虑到程序还有其他地方占用内存，可以认为重用机制降低了`Scrollview`超过50%的内存占用:

![优化后内存](https://user-gold-cdn.xitu.io/2018/2/16/1619ef2b22abe154?w=600&h=593&f=jpeg&s=35191)

不过这么做还是稍有不足，如果数据量比较大，频繁的重用`UIViewController`会导致多次`reloadData()`。切换页面的时候会稍有卡顿的感觉。也许是我哪里考虑欠周，欢迎指正。目前来看，重用机智更适合于呈现静态内容的`UIViewController`。

**项目地址**：[戳这里](https://github.com/Wl201314/MJianshu)，欢迎star。
