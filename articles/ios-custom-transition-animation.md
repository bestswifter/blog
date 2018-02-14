转场动画这事，说简单也简单，可以通过`presentViewController:animated:completion:`和`dismissViewControllerAnimated:completion:`这一组函数以模态视图的方式展现、隐藏视图。如果用到了`navigationController`，还可以调用`pushViewController:animated:`和`popViewController`这一组函数将新的视图控制器压栈、弹栈。

下图中所有转场动画都是自定义的动画，这些效果如果不用自定义动画则很难甚至无法实现：

![demo演示](http://images.bestswifter.com/CustomTransition/demo.gif)

由于录屏的原因，有些效果无法完全展现，比如它其实还支持横屏。

自定义转场动画的效果实现起来比较复杂，如果仅仅是拷贝一份能够运行的代码却不懂其中原理，就有可能带来各种隐藏的bug。本文由浅入深介绍下面几个知识：

1. 传统的基于闭包的实现方式及其缺点
2. 自定义present转场动画
3. 交互式(Interactive)转场动画
4. 转场协调器与UIModalPresentationCustom
5. UINavigationController转场动画

我为这篇教程制作了一个demo，您可以去在我的github上clone下来：[CustomTransition](https://github.com/bestswifter/MySampleCode/tree/master/CustomTransition)，如果觉得有帮助还望给个star以示支持。本文以Swift+纯代码实现，对应的OC+Storyboard版本在demo中也可以找到，那是苹果的官方示范代码，正确性更有保证。demo中用到了CocoaPods，您也许需要执行`pod install`命令并打开`.xcworkspace`文件。

在开始正式的教程前，您首先需要下载demo，在代码面前文字是苍白的，demo中包含的注释足以解释本文所有的知识点。其次，您还得了解这几个背景知识。

### From和To

在代码和文字中，经常会出现`fromView`和`toView`。如果错误的理解它们的含义会导致动画逻辑完全错误。`fromView`表示当前视图，`toView`表示要跳转到的视图。如果是从A视图控制器present到B，则A是from，B是to。从B视图控制器dismiss到A时，B变成了from，A是to。用一张图表示：

![from和to](http://images.bestswifter.com/CustomTransition/fromto.png)

### Presented和Presenting

这也是一组相对的概念，它容易与`fromView`和`toView`混淆。简单来说，它不受present或dismiss的影响，如果是从A视图控制器present到B，那么A总是B的`presentingViewController`,B总是A的`presentedViewController`。

### modalPresentationStyle

这是一个枚举类型，表示present时动画的类型。其中可以自定义动画效果的只有两种：`FullScreen `和`Custom`，两者的区别在于`FullScreen `会移除`fromView`，而`Custom`不会。比如文章开头的gif中，第三个动画效果就是`Custom`。

# 基于block的动画

最简单的转场动画是使用`transitionFromViewController`方法：

![传统的转场动画实现](http://images.bestswifter.com/CustomTransition/blockbased_animation.png)

这个方法虽然已经过时，但是对它的分析有助于后面知识的理解。它一共有6个参数，前两个表示从哪个VC开始，跳转到哪个VC，中间两个参数表示动画的时间和选项。最后两个参数表示动画的具体实现细节和回调闭包。

这六个参数其实就是一次转场动画所必备的六个元素。它们可以分为两组，前两个参数为一组，表示页面的跳转关系，后面四个为一组，表示动画的执行逻辑。

这个方法的缺点之一是可自定义程度不高(在后面您会发现能自定义的不仅仅是动画方式)，另一个缺点则是重用性不好，也可以说是耦合度比较大。

在最后两个闭包参数中，可以预见的是`fromViewController`和`toViewController`参数都会被用到，而且他们是动画的关键。假设视图控制器A可以跳转到B、C、D、E、F，而且跳转动画基本相似，您会发现`transitionFromViewController`方法要被复制多次，每次只会修改少量内容。

# 自定义present转场动画

出于解耦和提高可自定义程度的考虑，我们来学习转场动画的正确使用姿势。

首先要了解一个关键概念：转场动画代理，它是一个实现了`UIViewControllerTransitioningDelegate`协议的对象。我们需要自己实现这个对象，它的作用是为UIKit提供以下几个对象中的一个或多个：

1. Animator：
	
	它是实现了`UIViewControllerAnimatedTransitioning `协议的对象，用于控制动画的持续时间和动画展示逻辑，代理可以为present和dismiss过程分别提供Animator，也可以提供同一个Animator。
	
2. 交互式Animator：和Animator类似，不过它是交互式的，后面会有详细介绍
3. Presentation控制器：

	它可以对present过程更加彻底的自定义，比如修改被展示视图的大小，新增自定义视图等，后面会有详细介绍。
	
![转场动画代理](http://images.bestswifter.com/CustomTransition/delegate.png)
	
在这一小节中，我们首先介绍最简单的Animator。回顾一下转场动画必备的6个元素，它们被分为两组，彼此之间没有关联。Animator的作用等同于第二组的四个元素，也就是说对于同一个Animator，可以适用于A跳转B，也可以适用于A跳转C。它表示一种通用的页面跳转时的动画逻辑，不受限于具体的视图控制器。

如果您读懂了这段话，整个自定义的转场动画逻辑就很清楚了，以视图控制器A跳转到B为例：

1. 创建动画代理，在事情比较简单时，A自己就可以作为代理
2. 设置B的transitioningDelegate为步骤1中创建的代理对象
3. 调用`presentViewController:animated:completion:`并把参数animated设置为true
4. 系统会找到代理中提供的Animator，由Animator负责动画逻辑

用具体的例子解释就是：

```swift
// 这个类相当于A
class CrossDissolveFirstViewController: UIViewController, UIViewControllerTransitioningDelegate {
    // 这个对象相当于B
    crossDissolveSecondViewController.transitioningDelegate = self

    // 点击按钮触发的函数
    func animationButtonDidClicked() {
        self.presentViewController(crossDissolveSecondViewController, animated: true, completion: nil)
    }

    // 下面这两个函数定义在UIViewControllerTransitioningDelegate协议中
    // 用于为 present 和 dismiss 提供 animator
    func animationControllerForPresentedController(presented: UIViewController, presentingController presenting: UIViewController, sourceController source: UIViewController) -> UIViewControllerAnimatedTransitioning? {
      // 也可以使用CrossDissolveAnimator，动画效果各有不同
      // return CrossDissolveAnimator()
      return HalfWaySpringAnimator()
    }

    func animationControllerForDismissedController(dismissed: UIViewController) -> UIViewControllerAnimatedTransitioning? {
        return CrossDissolveAnimator()
    }
}
```

动画的关键在于animator如何实现，它实现了`UIViewControllerAnimatedTransitioning` 协议，至少需要实现两个方法，我建议您仔细阅读`animateTransition`方法中的注释，它是整个动画逻辑的核心：

```swift
class HalfWaySpringAnimator: NSObject, UIViewControllerAnimatedTransitioning {
    /// 设置动画的持续时间
    func transitionDuration(transitionContext: UIViewControllerContextTransitioning?) -> NSTimeInterval {
        return 2
    }

    /// 设置动画的进行方式，附有详细注释，demo中其他地方的这个方法不再解释
    func animateTransition(transitionContext: UIViewControllerContextTransitioning) {
        let fromViewController = transitionContext.viewControllerForKey(UITransitionContextFromViewControllerKey)
        let toViewController = transitionContext.viewControllerForKey(UITransitionContextToViewControllerKey)
        let containerView = transitionContext.containerView()

        // 需要关注一下 from/to 和 presented/presenting 的关系
        // For a Presentation:
        //      fromView = The presenting view.
        //      toView   = The presented view.
        // For a Dismissal:
        //      fromView = The presented view.
        //      toView   = The presenting view.

        var fromView = fromViewController?.view
        var toView = toViewController?.view

        // iOS8引入了viewForKey方法，尽可能使用这个方法而不是直接访问controller的view属性
        // 比如在form sheet样式中，我们为presentedViewController 的 view 添加阴影或其他decoration，animator 会对整个 decoration view
        // 添加动画效果，而此时presentedViewController 的 view 只是 decoration view 的一个子视图
        if transitionContext.respondsToSelector(Selector("viewForKey:")) {
            fromView = transitionContext.viewForKey(UITransitionContextFromViewKey)
            toView = transitionContext.viewForKey(UITransitionContextToViewKey)
        }

        // 我们让 toview 的 origin.y 在屏幕的一半处，这样它从屏幕的中间位置弹起而不是从屏幕底部弹起，弹起过程中逐渐变为不透明
        toView?.frame = CGRectMake(fromView!.frame.origin.x, fromView!.frame.maxY / 2, fromView!.frame.width, fromView!.frame.height)
        toView?.alpha = 0.0

        // 在present和，dismiss时，必须将 toview 添加到视图层次中
        containerView?.addSubview(toView!)

        let transitionDuration = self.transitionDuration(transitionContext)
        // 使用spring动画，有弹簧效果，动画结束后一定要调用completeTransition方法
        UIView.animateWithDuration(transitionDuration, delay: 0, usingSpringWithDamping: 0.6, initialSpringVelocity: 0, options: .CurveLinear, animations: { () -> Void in
            toView!.alpha = 1.0     // 逐渐变为不透明
            toView?.frame = transitionContext.finalFrameForViewController(toViewController!)    // 移动到指定位置
        }) { (finished: Bool) -> Void in
          let wasCancelled = transitionContext.transitionWasCancelled()
          transitionContext.completeTransition(!wasCancelled)
        }
    }
}
```

`animateTransition`方法的核心则是从转场动画上下文获取必要的信息以完成动画。上下文是一个实现了`UIViewControllerContextTransitioning`的对象，它的作用在于为`animateTransition`方法提供必备的信息。您不应该缓存任何关于动画的信息，而是应该总是从转场动画上下文中获取(比如fromView 和 toView)，这样可以保证总是获取到最新的、正确的信息。

![转场动画上下文](http://images.bestswifter.com/CustomTransition/context.png)

获取到足够信息后，我们调用`UIView.animateWithDuration `方法把动画交给Core Animation处理。千万不要忘记在动画调用结束后，执行`completeTransition `方法。

本节的知识在Demo的**Cross Dissolve**文件夹中有详细的代码。其中有两个animator文件，这说明我们可以为present和dismiss提供同一个animator，或者分别提供各自对应的animator。如果两者动画效果类似，您可以共用同一个animator，惟一的区别在于：

1. present时，要把`toView`加入到container的视图层级。
2. dismiss时，要把`fromView`从container的视图层级中移除。

如果您被前面这一大段代码和知识弄晕了，或者暂时用不到这些具体的知识，您至少需要记住自定义动画的基本原理和流程：

1. 设置将要跳转到的视图控制器(`presentedViewController`)的`transitioningDelegate`
2. 充当代理的对象可以是源视图控制器(`presentingViewController`)，也可以是自己创建的对象，它需要为转场动画提供一个animator对象。
3. animator对象的`animateTransition `是整个动画的核心逻辑。

# 交互式(Interactive)转场动画

刚刚我们说到，设置了`toViewController`的`transitioningDelegate`属性并且present时，UIKit会从代理处获取animator，其实这里还有一个细节：UIKit还会调用代理的`interactionControllerForPresentation: `方法来获取交互式控制器，如果得到了nil则执行非交互式动画，这就回到了上一节的内容。

如果获取到了不是nil的对象，那么UIKit不会调用animator的`animateTransition `方法，而是调用交互式控制器(还记得前面介绍动画代理的示意图么，交互式动画控制器和animator是平级关系)的`startInteractiveTransition:`方法。

所谓的交互式动画，通常是基于手势驱动，产生一个动画完成的百分比来控制动画效果(文章开头的gif中第二个动画效果)。整个动画不再是一次性、连贯的完成，而是在任何时候都可以改变百分比甚至取消。这需要一个实现了`UIPercentDrivenInteractiveTransition `协议的交互式动画控制器和animator协同工作。这看上去是一个非常复杂的任务，但UIKit已经封装了足够多细节，我们只需要在交互式动画控制器和中定义一个时间处理函数(比如处理滑动手势)，然后在接收到新的事件时，计算动画完成的百分比并且调用`updateInteractiveTransition`来更新动画进度即可。

用下面这段代码简单表示一下整个流程(删除了部分细节和注释，请不要以此为正确参考)，完整的代码请参考demo中的**Interactivity**文件夹：

```swift
// 这个相当于fromViewController
class InteractivityFirstViewController: UIViewController {
    // 这个相当于toViewController
    lazy var interactivitySecondViewController: InteractivitySecondViewController = InteractivitySecondViewController()
    // 定义了一个InteractivityTransitionDelegate类作为代理
    lazy var customTransitionDelegate: InteractivityTransitionDelegate = InteractivityTransitionDelegate()

    override func viewDidLoad() {
        super.viewDidLoad()
        setupView() // 主要是一些UI控件的布局，可以无视其实现细节

        /// 设置动画代理，这个代理比较复杂，所以我们新建了一个代理对象而不是让self作为代理
        interactivitySecondViewController.transitioningDelegate = customTransitionDelegate
    }

    // 触发手势时，也会调用animationButtonDidClicked方法
    func interactiveTransitionRecognizerAction(sender: UIScreenEdgePanGestureRecognizer) {
        if sender.state == .Began {
            self.animationButtonDidClicked(sender)
        }
    }

    func animationButtonDidClicked(sender: AnyObject) {
      self.presentViewController(interactivitySecondViewController, animated: true, completion: nil)
    }
}
```

非交互式的动画代理只需要为present和dismiss提供animator即可，但是在交互式的动画代理中，还需要为present和dismiss提供交互式动画控制器：

```swift
class InteractivityTransitionDelegate: NSObject, UIViewControllerTransitioningDelegate {
    func animationControllerForPresentedController(presented: UIViewController, presentingController presenting: UIViewController, sourceController source: UIViewController) -> UIViewControllerAnimatedTransitioning? {
        return InteractivityTransitionAnimator(targetEdge: targetEdge)
    }

    func animationControllerForDismissedController(dismissed: UIViewController) -> UIViewControllerAnimatedTransitioning? {
        return InteractivityTransitionAnimator(targetEdge: targetEdge)
    }

    /// 前两个函数和淡入淡出demo中的实现一致
    /// 后两个函数用于实现交互式动画

    func interactionControllerForPresentation(animator: UIViewControllerAnimatedTransitioning) -> UIViewControllerInteractiveTransitioning? {
        return TransitionInteractionController(gestureRecognizer: gestureRecognizer, edgeForDragging: targetEdge)
    }

    func interactionControllerForDismissal(animator: UIViewControllerAnimatedTransitioning) -> UIViewControllerInteractiveTransitioning? {
      return TransitionInteractionController(gestureRecognizer: gestureRecognizer, edgeForDragging: targetEdge)
    }
}
```

animator中的代码略去，它和非交互式动画中的animator类似。因为交互式的动画只是一种锦上添花，它必须支持非交互式的动画，比如这个例子中，点击按钮依然出发的是非交互式的动画，只是手势滑动才会触发交互式动画。

```swift
class TransitionInteractionController: UIPercentDrivenInteractiveTransition {
    /// 当手势有滑动时触发这个函数
    func gestureRecognizeDidUpdate(gestureRecognizer: UIScreenEdgePanGestureRecognizer) {
        switch gestureRecognizer.state {
            case .Began: break
            case .Changed: self.updateInteractiveTransition(self.percentForGesture(gestureRecognizer))  //手势滑动，更新百分比
            case .Ended:    // 滑动结束，判断是否超过一半，如果是则完成剩下的动画，否则取消动画
                if self.percentForGesture(gestureRecognizer) >= 0.5 {
                    self.finishInteractiveTransition()
                }
            else {
                self.cancelInteractiveTransition()
            }
            default: self.cancelInteractiveTransition()
        }
    }

    private func percentForGesture(gesture: UIScreenEdgePanGestureRecognizer) -> CGFloat {
        let percent = 根据gesture计算得出
        return percent
    }
}
```

交互式动画是在非交互式动画的基础上实现的，我们需要创建一个继承自`UIPercentDrivenInteractiveTransition` 类型的子类，并且在动画代理中返回这个类型的实例对象。

在这个类型中，监听手势(或者下载进度等等)的时间变化，然后调用`percentForGesture`方法更新动画进度即可。

# 转场协调器与UIModalPresentationCustom

在进行转场动画的同时，您还可以进行一些同步的，额外的动画，比如文章开头gif中的第三个例子。`presentedView`和`presentingView`可以更改自身的视图层级，添加额外的效果(阴影，圆角)。UIKit使用转成协调器来管理这些额外的动画。您可以通过需要产生动画效果的视图控制器的`transitionCoordinator`属性来获取转场协调器，转场协调器只在转场动画的执行过程中存在。

![转场动画协调器](http://images.bestswifter.com/CustomTransition/coordinator.png)

想要完成gif中第三个例子的效果，我们还需要使用`UIModalPresentationStyle.Custom`来代替`.FullScreen`。因为后者会移除`fromViewController`，这显然不符合需求。

当present的方式为`.Custom`时，我们还可以使用`UIPresentationController`更加彻底的控制转场动画的效果。一个 presentation controller具备以下几个功能：

1. 设置`presentedViewController`的视图大小
2. 添加自定义视图来改变`presentedView`的外观
3. 为任何自定义的视图提供转场动画效果
4. 根据size class进行响应式布局

您可以认为，`. FullScreen `以及其他present风格都是swift为我们实现提供好的，它们是`.Custom`的特例。而`.Custom`允许我们更加自由的定义转场动画效果。

`UIPresentationController`提供了四个函数来定义present和dismiss动画开始前后的操作：

1. `presentationTransitionWillBegin`: present将要执行时
2. `presentationTransitionDidEnd`：present执行结束后
3. `dismissalTransitionWillBegin`：dismiss将要执行时
4. `dismissalTransitionDidEnd`：dismiss执行结束后

下面的代码简要描述了gif中第三个动画效果的实现原理，您可以在demo的**Custom Presentation**文件夹下查看完成代码：

```swift
// 这个相当于fromViewController
class CustomPresentationFirstViewController: UIViewController {
    // 这个相当于toViewController
    lazy var customPresentationSecondViewController: CustomPresentationSecondViewController = CustomPresentationSecondViewController()
    // 创建PresentationController
    lazy var customPresentationController: CustomPresentationController = CustomPresentationController(presentedViewController: self.customPresentationSecondViewController, presentingViewController: self)

    override func viewDidLoad() {
        super.viewDidLoad()
        setupView() // 主要是一些UI控件的布局，可以无视其实现细节

        // 设置转场动画代理
        customPresentationSecondViewController.transitioningDelegate = customPresentationController
    }

    override func didReceiveMemoryWarning() {
        super.didReceiveMemoryWarning()
        // Dispose of any resources that can be recreated.
    }

    func animationButtonDidClicked() {
        self.presentViewController(customPresentationSecondViewController, animated: true, completion: nil)
    }
}
```

重点在于如何实现 `CustomPresentationController` 这个类：

```swift
class CustomPresentationController: UIPresentationController, UIViewControllerTransitioningDelegate {
    var presentationWrappingView: UIView?  // 这个视图封装了原视图，添加了阴影和圆角效果
    var dimmingView: UIView? = nil  // alpha为0.5的黑色蒙版

    // 告诉UIKit为哪个视图添加动画效果
    override func presentedView() -> UIView? {
        return self.presentationWrappingView
    }
}

// 四个方法自定义转场动画发生前后的操作
extension CustomPresentationController {
    override func presentationTransitionWillBegin() {
        // 设置presentationWrappingView和dimmingView的UI效果
        let transitionCoordinator = self.presentingViewController.transitionCoordinator()
        self.dimmingView?.alpha = 0
        // 通过转场协调器执行同步的动画效果
        transitionCoordinator?.animateAlongsideTransition({ (context: UIViewControllerTransitionCoordinatorContext) -> Void in
            self.dimmingView?.alpha = 0.5
        }, completion: nil)
    }

    /// present结束时，把dimmingView和wrappingView都清空，这些临时视图用不到了
    override func presentationTransitionDidEnd(completed: Bool) {
        if !completed {
            self.presentationWrappingView = nil
            self.dimmingView = nil
        }
    }

    /// dismiss开始时，让dimmingView完全透明，这个动画和animator中的动画同时发生
    override func dismissalTransitionWillBegin() {
        let transitionCoordinator = self.presentingViewController.transitionCoordinator()
        transitionCoordinator?.animateAlongsideTransition({ (context: UIViewControllerTransitionCoordinatorContext) -> Void in
          self.dimmingView?.alpha = 0
        }, completion: nil)
    }

    /// dismiss结束时，把dimmingView和wrappingView都清空，这些临时视图用不到了
    override func dismissalTransitionDidEnd(completed: Bool) {
        if completed {
            self.presentationWrappingView = nil
            self.dimmingView = nil
        }
    }
}

extension CustomPresentationController {
}
```

除此以外，这个类还要处理子视图布局相关的逻辑。它作为动画代理，还需要为动画提供animator对象，详细代码请在demo的**Custom Presentation**文件夹下阅读。

# UINavigationController转场动画

到目前为止，所有转场动画都是适用于 present 和 dismiss 的，其实`UINavigationController` 也可以自定义转场动画。两者是平行关系，很多都可以类比过来：

```swift
class FromViewController: UIViewController, UINavigationControllerDelegate {
    let toViewController: ToViewController = ToViewController()

    override func viewDidLoad() {
        super.viewDidLoad()
        setupView() // 主要是一些UI控件的布局，可以无视其实现细节

        self.navigationController.delegate = self
    }
}
```

与present/dismiss不同的时，现在视图控制器实现的是`UINavigationControllerDelegate` 协议，让自己成为`navigationController` 的代理。这个协议类似于此前的`UIViewControllerTransitioningDelegate` 协议。

`FromViewController` 实现`UINavigationControllerDelegate` 协议的具体操作如下：

```swift
func navigationController(navigationController: UINavigationController, animationControllerForOperation operation: UINavigationControllerOperation, fromViewController fromVC: UIViewController, toViewController toVC: UIViewController) -> UIViewControllerAnimatedTransitioning? {
    if operation == .Push {
        return PushAnimator()
    }
    if operation == .Pop {
        return PopAnimator()
    }
    return nil;
}
```

至于animator，就和此前没有任何区别了。可见，一个封装得很好的animator，不仅能在present/dismiss时使用，甚至还可以在push/pop时使用。

UINavigationController也可以添加交互式转场动画，原理也和此前类似。

# 总结

对于非交互式动画，需要设置`presentedViewController` 的`transitioningDelegate` 属性，这个代理需要为 present 和 dismiss 提供 animator。在 animator 中规定了动画的持续时间和表现逻辑。

对于交互式动画，需要在此前的基础上，由`transitioningDelegate` 属性提供交互式动画控制器。在控制器中进行事件处理，然后更新动画完成进度。

对于自定义动画，可以通过`UIPresentationController` 中的四个函数自定义动画执行前后的效果，可以修改`presentedViewController` 的大小、外观并同步执行其他的动画。

自定义动画的水还是比较深，本文仅适合做入门学习用，欢迎互相交流。