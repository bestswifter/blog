# 序言

### 本文将简要讨论一下几个问题：

1. <font color = "rgb(226,238,250)">`loadView`</font>、<font color = "rgb(226,238,250)">`viewDidLoad`</font>、<font color = "rgb(226,238,250)">`viewDidAppear`</font>、<font color = "rgb(226,238,250)">`initWithNibName`</font>、<font color = "rgb(226,238,250)">`awakeFromNib`</font>等经常出现在UIViewController中的方法介绍。
2. 这些方法分别用来作哪些工作，换言之，创建自定义的View时代码放到以上哪个方法中。
3. 一个UIView的生命周期是怎样的。以上几个方法的调用顺序如何。
4. 通过IB和代码加载视图，有什么区别

##### 文章主要参考官方和文档和StackOVerFlow有关问题整理得出，由于水平有限，如有错误之处请及时与我联系。

# UIViewController视图管理

### 视图层次和根视图

每个视图控制器都维护一个<font color=red>**视图层次(view hierarchy)**</font>。

因为每个视图都有自己的**子视图**，这个**视图层次**其实也可以理解为一棵树状的数据结构。而树的根节点，也就是根视图(root view)，在UIViewController中以<font color = "rgb(226,238,250)">`view`</font>属性。它可以被看做是其他所有子视图的容器。

---
### 视图的加载方式

UIViewController采用懒加载的方式，也就是说第一次访问到<font color = "rgb(226,238,250)">`view`</font>属性时才会加载或创建它。由于视图由视图控制器管理，所以讨论视图的加载方式时，主要讨论视图控制器的加载方式。

* 通过Storyboard加载：这是苹果推荐的方式，也是未来的趋势。			
	
	 通过这种方式创建<font color = "rgb(226,238,250)">`UIViewController `</font>对象的话，首先生成<font color = "rgb(226,238,250)">`UIStoryboard`</font>类型的对象，然后调用这个对象的<font color = "rgb(226,238,250)">`instantiateViewControllerWithIdentifier: `</font>方法

* 通过Nib文件加载：

	Nib文件其实就是xib文件，Storyboard相当于是聚合了多个nib文件，并且添加了对不同的<font color = "rgb(226,238,250)">`UIViewController `</font>之间的segue和relationship的管理。但总的实现原理非常类似
	
	通过这种方式加载视图,需要调用<font color = "rgb(226,238,250)">`UIViewController `</font>类的<font color = "rgb(226,238,250)">`initWithNibName:bundle:  `</font>方法
* 通过loadview方法加载：

	这就是通过代码加载。这需要我们在<font color = "rgb(226,238,250)">`loadView`</font>	方法中，通过编程创建自己的**视图层次**，并且把把根视图赋值给<font color = "rgb(226,238,250)">`UIViewController `</font>的<font color = "rgb(226,238,250)">`view `</font>属性。
	
因此，通过代码自定义View的时候，<font color = "rgb(226,238,250)">`loadView`</font>	方法大概是这样的：
	
```Objective-C
- (void)loadView{
	self.view = [[XXXView alloc] init];
}
```

---
### 处理视图相关通知

当视图的可见性发生变化时，视图控制器会自动调用一系列方法来响应变化。

所有可能的状态、方法和状态之间的转换关系在下图中被明确标出。

![](http://7xonij.com1.z0.glb.clouddn.com/UIViewLifeCircle/StateTransitionspng.png)

可以看到每一个will方法都有自己对应的did方法。但是如果我们在will方法中开始一个任务，不仅要在对应的did方法中结束它，还要考虑到和这个will方法相反的那个will方法（注意到Appearing和Disappearing这两个状态是可以互相转化的）

---
# 在运行时展示View

UIKit极大的简化了加载和展示View的过程，它大概会按照以下顺序执行一些任务：

1. 通过storyboard文件中的信息实例化视图
2. 连接outlet和action
3. 把根视图赋值给<font color = "rgb(226,238,250)">`UIViewController `</font>的<font color = "rgb(226,238,250)">`view `</font>属性（其实就是调用<font color = "rgb(226,238,250)">`loadView`</font>	方法）
4. 调用<font color = "rgb(226,238,250)">`UIViewController `</font>的<font color = "rgb(226,238,250)">`awakeFromNib `</font>方法。要注意，在调用方法前，</font>的<font color = "rgb(226,238,250)">`trait collecion`</font>为空且子视图的位置可能不正确
5. 调用<font color = "rgb(226,238,250)">`UIViewController `</font>的<font color = "rgb(226,238,250)">`viewDidLoad `</font>方法。

此时已经完成了视图的加载工作，在展示到屏幕之前，还有以下几个步骤：

6. 调用<font color = "rgb(226,238,250)">`UIViewController `</font>的<font color = "rgb(226,238,250)">`viewWillAppear `</font>方法。
7. 更新视图的布局
8. 把视图展示到屏幕上
9. 调用<font color = "rgb(226,238,250)">`UIViewController `</font>的<font color = "rgb(226,238,250)">`viewDidAppear `</font>方法。

### awakeFromNib方法

至此，第一个问题已经几乎解释完了，还剩一个<font color = "rgb(226,238,250)">`awakeFromNib `</font>方法。

我们已经知道，<font color = "rgb(226,238,250)">`awakeFromNib `</font>方法被调用时，所有视图的outlet和action已经连接，但还没有被确定。这个方法可以算作是和视图控制器的实例化配合在一起使用的，因为有些需要根据用户喜好来进行设置的内容，无法存在storyboard中，所以可以在<font color = "rgb(226,238,250)">`awakeFromNib `</font>方法中被加载进来。

<font color = "rgb(226,238,250)">`awakeFromNib `</font>方法在视图控制器的生命周期内只会被调用一次。因为它和视图控制器从nib文件中的解档密切相关，和view的关系却不大。

# 具体方法的解释

### loadView方法

当执行到<font color = "rgb(226,238,250)">`loadView `</font>方法时，视图控制器已经从nib文件中被解档并创建好了，接下来的任务主要是对view进行初始化。

<font color = "rgb(226,238,250)">`loadView `</font>方法在<font color = "rgb(226,238,250)">`UIViewController `</font>对象的<font color = "rgb(226,238,250)">`view `</font>属性被访问到且为空的时候调用。
这是它与<font color = "rgb(226,238,250)">`awakeFromNib `</font>方法的一个区别。假设我们在处理内存警告时释放<font color = "rgb(226,238,250)">`view `</font>属性（其实并不应该这么做，这里举个例子）：<font color = "rgb(226,238,250)">`self.view = nil `</font>。因此<font color = "rgb(226,238,250)">`loadView `</font>方法在视图控制器的生命周期内可能会被多次调用。

这个方法不应该被直接调用，而是由系统自动调用。它会加载或创建一个view并把它赋值给<font color = "rgb(226,238,250)">`UIViewController `</font>的<font color = "rgb(226,238,250)">`view `</font>属性。

在创建view的过程中，首先会根据<font color = "rgb(226,238,250)">`nibName `</font>去找对应的Nib文件然后加载。如果<font color = "rgb(226,238,250)">`nibName `</font>为空，或找不到对应的Nib文件，则会创建一个空视图(这种情况一般是纯代码，也就是为什么说代码构建View的时候，要重写<font color = "rgb(226,238,250)">`loadView`</font>	方法)。

注意在重写<font color = "rgb(226,238,250)">`loadView`</font>方法的时候，不要调用父类的方法。

---
### viewDidLoad方法
<font color = "rgb(226,238,250)">`loadView`</font>方法执行完之后，就会执行<font color = "rgb(226,238,250)">`viewDidLoad `</font>方法。此时整个**视图层次(view hierarchy)**已经被放到内存中。

无论是从nib文件加载，还是通过纯代码编写界面，<font color = "rgb(226,238,250)">`viewDidLoad `</font>方法都会执行。我们可以重写这个方法，对通过nib文件加载的view做一些其他的初始化工作。比如可以移除一些视图，修改约束，加载数据等。

---
### viewWillAppear和viewDidAppear方法

在视图加载完成，并即将显示在屏幕上时，会调用<font color = "rgb(226,238,250)">`viewWillAppear `</font>方法，在这个方法里，可以改变当前屏幕方向或状态栏的风格等。

当<font color = "rgb(226,238,250)">`viewWillAppear `</font>方法执行完后，系统会执行<font color = "rgb(226,238,250)">`viewDidAppear `</font>方法。在这个方法中，还可以对视图做一些关于展示效果方面的修改。


# 视图的生命历程

到目前为止，我们已经了解了每个方法的作用，接下来就把整个流程梳理一遍。

1. <font color = "rgb(226,238,250)">`-[ViewController  initWithCoder:]`或`-[ViewController  initWithNibName:Bundle]`</font>:首先从归档文件中加载<font color = "rgb(226,238,250)">`UIViewController `</font>对象。即使是纯代码，也会把nil作为参数传给后者。
2. <font color = "rgb(226,238,250)">`-[ViewController awakeFromNib]`</font>:作为第一个方法的助手，方便处理一些额外的设置。
3. <font color = "rgb(226,238,250)">`-[ViewController loadView]`</font>:创建或加载一个view并把它赋值给<font color = "rgb(226,238,250)">`UIViewController `</font>的<font color = "rgb(226,238,250)">`view `</font>属性
4. <font color = "rgb(226,238,250)">`-[ViewController viewDidLoad]`</font>:此时整个**视图层次(view hierarchy)**已经被放到内存中，可以移除一些视图，修改约束，加载数据等
5. <font color = "rgb(226,238,250)">`-[ViewController viewWillAppear:] `</font>:视图加载完成，并即将显示在屏幕上,还没有设置动画，可以改变当前屏幕方向或状态栏的风格等。
6. <font color = "rgb(226,238,250)">`-[ViewController viewWillLayoutSubviews]`</font>：即将开始子视图位置布局
7. <font color = "rgb(226,238,250)">`-[ViewController viewDidLayoutSubviews] `</font>：用于通知视图的位置布局已经完成
8. <font color = "rgb(226,238,250)">`-[ViewController viewDidAppear:] `</font>：视图已经展示在屏幕上，可以对视图做一些关于展示效果方面的修改。
9. <font color = "rgb(226,238,250)">`-[ViewController viewWillDisappear:]`</font>：视图即将消失
10. <font color = "rgb(226,238,250)">`-[ViewController viewDidDisappear:]`</font>：视图已经消失

如果考虑<font color = "rgb(226,238,250)">`UIViewController`</font>可能在某个时刻释放整个<font color = "rgb(226,238,250)">`view`</font>。那么再次加载视图时显然会从步骤3开始。因为此时的<font color = "rgb(226,238,250)">`UIViewController`</font>对象依然存在。

# 总结

1. 只有init系列的方法,如<font color = "rgb(226,238,250)">`initWithNibName`</font>需要自己调用，其他方法如<font color = "rgb(226,238,250)">`loadView`</font>和<font color = "rgb(226,238,250)">`awakeFromNib`</font>则是系统自动调用。而<font color = "rgb(226,238,250)">`viewWill/Did`</font>系列的方法则类似于回调和通知，也会被自动调用。
2. 纯代码写视图布局时需要注意，要手动调用<font color = "rgb(226,238,250)">`loadView`</font>方法，而且不要调用父类的<font color = "rgb(226,238,250)">`loadView`</font>方法。纯代码和用IB的区别仅存在于<font color = "rgb(226,238,250)">`loadView`</font>方法及其之前，编程时需要注意的也就是<font color = "rgb(226,238,250)">`loadView`</font>方法。
3. 除了<font color = "rgb(226,238,250)">`initWithNibName`</font>和<font color = "rgb(226,238,250)">`awakeFromNib`</font>方法是处理视图控制器外，其他方法都是处理视图。这两个方法在视图控制器的生命周期里只会调用一次。

# 参考资料

 1. [UIViewController Class Reference](https://developer.apple.com/library/ios/documentation/UIKit/Reference/UIViewController_Class/)
 2. [View Controller Programming Guide for iOS](https://developer.apple.com/library/ios/featuredarticles/ViewControllerPGforiPhoneOS/DefiningYourSubclass.html#//apple_ref/doc/uid/TP40007457-CH7-SW1)
 3. [NSObject UIKit Additions Reference](https://developer.apple.com/library/ios/documentation/UIKit/Reference/NSObject_UIKitAdditions/index.html#//apple_ref/occ/instm/NSObject/awakeFromNib)
 4. [Process of a UIViewController birth](http://stackoverflow.com/questions/5107604/can-somebody-explain-the-process-of-a-uiviewcontroller-birth-which-method-follo)
 5. [Which should I use, -awakeFromNib or -viewDidLoad?](http://stackoverflow.com/questions/377202/which-should-i-use-awakefromnib-or-viewdidload)