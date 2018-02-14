圆角（RounderCorner）是一种很常见的视图效果，相比于直角，它更加柔和优美，易于接受。但很多人并不清楚如何设置圆角的正确方式和原理。设置圆角会带来一定的性能损耗，如何提高性能是另一个需要重点讨论的话题。我查阅了一些现有的资料，收获良多的同时也发现了一些误导人错误。本文总结整理了一些知识点，概括如下：

* 设置圆角的正确姿势及其原理
* 设置圆角的性能损耗
* 其他设置圆角的方法，以及最优选择

我为本文制作了一个 demo，读者可以在我的 github 上 clone 下来：[CornerRadius](https://github.com/bestswifter/MySampleCode/tree/master/CornerRadius)，如果觉得有帮助还望给个star以示支持。项目由 Swift 实现，但请务必相信我即使你只会 Objective-C，也可以看懂它。因为其中的关键知识与 Swift 无关。

# 正确姿势

首先，我想要声明的一点是：

> **设置圆角很简单，它不会带来任何性能损耗**

因为这件事本来就很简单，它只需要一行代码：

```swift
view.layer.cornerRadius = 5
```

先别急着关掉网页，也别急着回复，我们让事实说话。打开 Instuments，选择 **Core Animation** 调试，你会发现既没有 Off-Screen Render，也没有降低帧数。关于使用 Instuments 分析应用，你可以参考我的这篇文章：[UIKit性能调优实战讲解](http://www.jianshu.com/p/619cf14640f3)。从截图中可以看到第三个棕色视图**确确实实**设置了圆角：

![圆角效果](http://images.bestswifter.com/corner.png)

不过查看一下代码可以发现，有一个 `UILabel` 也设置了圆角，但是没有表现出任何变化。关于这一点，你可以查看 `cornerRadius` 属性的注释：

> By default, the corner radius does not apply to the image in the layer’s contents property; it applies only to the background color and border of the layer. However, setting the masksToBounds property to true causes the content to be clipped to the rounded corners.

也就是说在默认情况下，这个属性只会影响视图的背景颜色和 border。对于 `UILabel` 这样内部还有子视图的控件就无能为力了。所以很多情况下我们会看到这样的代码：

```swift
label.layer.cornerRadius = 5
label.layer.masksToBounds = true
```

我们把第二行代码添加到 `CustomTableViewCell` 的构造方法中，再次运行 Instument，就可以看到圆角效果了。

# 性能损耗

如果你勾选上 **Color Offscreen-Rendered Yellow**，就会发现 label 的四周出现了黄色的标记，说明这里出现了离屏渲染。关于离屏渲染的介绍，同样可以参考：[UIKit性能调优实战讲解](http://www.jianshu.com/p/619cf14640f3)，就不在本文赘述了。

需要强调的一点是，**离屏渲染并非由设置圆角导致的！**通过控制变量的方法很容易得出这个结论，因为 UIView 只是设置了 `cornerRadius`，但它没有出现离屏渲染。某些比较权威的文章，比如 [Stackoverflow](http://stackoverflow.com/questions/13158796/what-triggers-offscreen-rendering-blending-and-layoutsubviews-in-ios) 和 [CodeReview](http://www.reviewcode.cn/article.html?reviewId=7) 都提到设置 `cornerRadius` 会导致离屏渲染从而影响性能，我想这实在是冤枉了可爱的 `cornerRadius` 变量，也误导了别人。

虽然设置 `masksToBounds` 会导致离屏渲染，从而影响性能，但是这个影响到底会有多大？在我的 iPhone6 上，即使出现了 17 个带有圆角的视图，滑动时的帧数依然在 58 - 59 fps 左右波动。

然而，这并非说明 iOS 9 做了什么特殊优化，或者是离屏渲染的影响不大，其主要原因在于**圆角不够多**。当我将一个 `UIImageView` 也设置成圆角，也就是屏幕上的圆角视图达到 34 个时，fps 大幅度下降，大约只有 33 左右。基本上已经达到了影响用户体验的范围。因此，一切不讲依据的优化都是耍流氓，如果你的圆角视图不多，cell 不复杂，就不要费力气折腾了。

# 高效地设置圆角

假设现在圆角视图非常多（比如在 UICollectionView 中），那么如何为视图高效的添加圆角呢？网上的教程大多没有说全，因为这个事要分两种情况考虑。为普通的 `UIView` 设置圆角，和为 `UIImageView` 设置圆角的原理截然不同。

有一种做法是这样的，这种写法试图实现 `cornerRadius = 3` 的效果：

```swift
override func drawRect(rect: CGRect) {
    let maskPath = UIBezierPath(roundedRect: rect,
                                byRoundingCorners: .AllCorners,
                                cornerRadii: CGSize(width: 3, height: 3))
    let maskLayer = CAShapeLayer()
    maskLayer.frame = self.bounds
    maskLayer.path = maskPath.CGPath
    self.layer.mask = maskLayer
}
```

**不过这是一种错的离谱的写法！**

首先，我们应该尽量避免重写 `drawRect` 方法。不恰当的使用这个方法会导致内存暴增。举个例子，iPhone6 上与屏幕等大的 `UIView`，即使重写一个空的 `drawRect` 方法，它也至少占用 `750 * 1134 * 4 字节 ≈ 3.4 Mb` 的内存。在 [内存恶鬼drawRect](http://bihongbo.com/2016/01/03/memoryGhostdrawRect/) 及其后续中，作者详细介绍了其中原理，据他测试，在 iPhone6 上空的、与屏幕等大的视图重写 `drawRect` 方法会消耗 5.2 Mb 内存。总之，能避免重写 `drawRect` 方法就尽可能避免。

其次，这种方法本质上是用遮罩层 `mask` 来实现，因此同样无可避免的会导致离屏渲染。我试着将此前 34 个视图的圆角改用这种方法实现，结果 fps 掉到 11 左右。已经属于卡出翔的节奏了。

忘掉这种写法吧，下面介绍正确的高效设置圆角的姿势。

## 为 UIView 添加圆角

这种做法的原理是手动画出圆角。虽然我们之前说过，为普通的视图直接设置 `cornerRadius` 属性即可。但万一不可避免的需要使用 `masksToBounds`，就可以使用下面这种方法，它的核心代码如下：

```swift
func kt_drawRectWithRoundedCorner(radius radius: CGFloat,
                                  borderWidth: CGFloat,
                                  backgroundColor: UIColor,
                                  borderColor: UIColor) -> UIImage {    
    UIGraphicsBeginImageContextWithOptions(sizeToFit, false, UIScreen.mainScreen().scale)
    let context = UIGraphicsGetCurrentContext()
    
    CGContextMoveToPoint(context, 开始位置);  // 开始坐标右边开始
    CGContextAddArcToPoint(context, x1, y1, x2, y2, radius);  // 这种类型的代码重复四次
    
    CGContextDrawPath(UIGraphicsGetCurrentContext(), .FillStroke)
    let output = UIGraphicsGetImageFromCurrentImageContext();
    UIGraphicsEndImageContext();
    return output
}
```

这个方法返回的是 `UIImage`，也就是说我们利用 Core Graphics 自己画出了一个圆角矩形。除了一些必要的代码外，最核心的就是 `CGContextAddArcToPoint ` 函数。它中间的四个参数表示曲线的起点和终点坐标，最后一个参数表示半径。调用了四次函数后，就可以画出圆角矩形。最后再从当前的绘图上下文中获取图片并返回。

有了这个图片后，我们创建一个 `UIImageView` 并插入到视图层级的底部：

```swift
extension UIView {
    func kt_addCorner(radius radius: CGFloat,
                      borderWidth: CGFloat,
                      backgroundColor: UIColor,
                      borderColor: UIColor) {
        let imageView = UIImageView(image: kt_drawRectWithRoundedCorner(radius: radius,
                                    borderWidth: borderWidth,
                                    backgroundColor: backgroundColor,
                                    borderColor: borderColor))
        self.insertSubview(imageView, atIndex: 0)
    }
}
```

完整的代码可以在项目中找到，使用时，你只需要这样写：

```swift
let view = UIView(frame: CGRectMake(1,2,3,4))
view.kt_addCorner(radius: 6)
```

## 为 UIImageView 添加圆角

相比于上面一种实现方法，为 `UIImageView` 添加圆角更为常用。它的实现思路是直接截取图片：

```swift
extension UIImage {
    func kt_drawRectWithRoundedCorner(radius radius: CGFloat, _ sizetoFit: CGSize) -> UIImage {
        let rect = CGRect(origin: CGPoint(x: 0, y: 0), size: sizetoFit)

        UIGraphicsBeginImageContextWithOptions(rect.size, false, UIScreen.mainScreen().scale)
        CGContextAddPath(UIGraphicsGetCurrentContext(),
            UIBezierPath(roundedRect: rect, byRoundingCorners: UIRectCorner.AllCorners,
                cornerRadii: CGSize(width: radius, height: radius)).CGPath)
        CGContextClip(UIGraphicsGetCurrentContext())

        self.drawInRect(rect)
        CGContextDrawPath(UIGraphicsGetCurrentContext(), .FillStroke)
        let output = UIGraphicsGetImageFromCurrentImageContext();
        UIGraphicsEndImageContext();

        return output
    }
}
```

圆角路径直接用贝塞尔曲线绘制，一个意外的 bonus 是还可以选择哪几个角有圆角效果。这个函数的效果是将原来的 `UIImage` 剪裁出圆角。配合着这函数，我们可以为 UIImageView 拓展一个设置圆角的方法：

```swift
extension UIImageView {
    /**
     / !!!只有当 imageView 不为nil 时，调用此方法才有效果

     :param: radius 圆角半径
     */
    override func kt_addCorner(radius radius: CGFloat) {
        self.image = self.image?.kt_drawRectWithRoundedCorner(radius: radius, self.bounds.size)
    }
}
```

完整的代码可以在项目中找到，使用时，你只需要这样写：

```swift
let imageView = let imgView1 = UIImageView(image: UIImage(name: ""))
imageView.kt_addCorner(radius: 6)
```

### 提醒

无论使用上面哪种方法，你都需要小心使用背景颜色。因为此时我们没有设置 `masksToBounds`，因此超出圆角的部分依然会被显示。因此，你不应该再使用背景颜色，可以在绘制圆角矩形时设置填充颜色来达到类似效果。

在为 `UIImageView` 添加圆角时，请确保 `image` 属性不是 `nil`，否则这个设置将会无效。

# 实战测试

回到 demo 中，测试一下刚刚定义的这两个设置圆角的方法。首先在 `setupContent` 方法中把这两行代码的注释取消掉：

```swift
imgView1.kt_addCorner(radius: 5)
imgView2.kt_addCorner(radius: 5)

```

然后使用自定义的方法为 label 和 view 设置圆角：

```swift
view.kt_addCorner(radius: 6)
label.kt_addCorner(radius: 6)
```

现在，我们不仅成功的添加了圆角效果，同时还保证了性能不受影响：

![性能测试](http://images.bestswifter.com/CornerRadius/fps.jpeg)

# 总结

1. 如果能够只用 `cornerRadius` 解决问题，就不用优化。
2. 如果必须设置 `masksToBounds`，可以参考圆角视图的数量，如果数量较少（一页只有几个）也可以考虑不用优化。
3. `UIImageView` 的圆角通过直接截取图片实现，其它视图的圆角可以通过 Core Graphics 画出圆角矩形实现。

# 参考资料

1. [小心别让圆角成了你列表的帧数杀手](http://www.cocoachina.com/ios/20150803/12873.html)
2. [关于性能的一些问题](http://www.reviewcode.cn/article.html?reviewId=7)
