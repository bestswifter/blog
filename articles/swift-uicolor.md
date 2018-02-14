我为这篇文章制作了一个demo，已经上传到我的GitHub：[KTColor](https://github.com/bestswifter/MySampleCode/tree/master/KtColor)，如果觉得有帮助还望给个star以示支持。

`UIColor` 提供了几个默认的颜色，要想创建除此以外的颜色，一般是通过RGB和alpha值创建(十六进制的颜色其实也是被转换成RGB)。在 Objective-C 中，这可以通过自定义宏来完成，在 Swift 中，我们可以利用 Swift 的一些语法特性来简化创建 `UIColor` 对象的过程。我想，最理想的解决方案应该是这样：

```swift
override func viewDidLoad() {
    super.viewDidLoad()

    self.view.backgroundColor = "224, 222, 255"
}
```

# 变通方案

然而很不幸的是，在目前的 Swift 版本(2.1)中，这种写法暂时无法实现。据我所知，Swift3.0 也不支持这种写法，原因会在稍后分析。目前，我们可以使用两种变通方案：

```swift
self.view.backgroundColor = "224, 222, 255".ktColor    // 方案1
self.view.backgroundColor = "224, 222, 255" as KtColor    // 方案2
```

两者写法类似，但实现原理实际上完全不同。第一种方案是通过拓展 `String` 类型实现的，第二种方案则是通过继承 `UIColor` 实现。

方案1有更好的代码提示，但它对 `String` 类型作了修改，我的demo中有完整的实现，它支持以下输入：

```swift
self.view.backgroundColor = "224, 222, 255, 0.5".ktcolor // 这个是完整版
self.view.backgroundColor = "224, 222, 255".ktcolor // alpha值默认为1
self.view.backgroundColor = "224,222,255".ktcolor // 可以用逗号分割
self.view.backgroundColor = "224 222 255".ktcolor // 可以用空格分割
self.view.backgroundColor = "#DC143C".ktcolor  // 可以使用16进制数字
self.view.backgroundColor = "#dc143c".ktcolor  // 字母可以小写
self.view.backgroundColor = "SkyBlue".ktcolor  // 可以直接使用颜色的英文名字
```

虽然方案2不会对现有代码做修改，但它并不适用于所有系统类型，比如 `NSDate` 或 `NSURL` 类型，出于这种考虑，demo中仅实现了关键逻辑。但这种实现方法最接近于理想的解决方案，一旦时机合适，我们就可以去掉丑陋的 `as KtColor`。

# 拓展字符串

第一种方案通过拓展 `String` 类型实现，它添加了一个 `ktcolor` 计算属性，主要涉及到字符串的分割与处理，还有一些容错、判断等，这些就不是本文的重点了，如果有兴趣，读者可以通过阅读源码获得更加深入的了解。

这种方案的好处在于它还适用于 `NSDate`、`NSURL`等类型。比如，下面的代码可以通过类似的技术实现：

```swift
let date = "2016-02-17 24:00:00".ktdate
let url = "http://bestswifter.com".kturl
```

不过，方案一选择的技术注定了它没有再简化的空间了。如果不能显著的减少代码量，它就没有理由取代原生的方案。

# 字符串字面量

方案二和理想方案采用的都是同一个思路：“利用字符串字面量创建对象”。在我的[这篇文章](http://www.jianshu.com/p/07cf2a6ad917)中对此有比较详细的解释。

简单来说，我们要做的只是为 `UIColor` 类型添加如下的拓展：

```swift
extension UIColor: StringLiteralConvertible {
    public init(stringLiteral value: String) {
    	//这里的数字是随便写的，实际上需要解析字符串
        self.init(red: 0.5, green: 0.8, blue: 0.25, alpha: 1)
    }
    
    public init(extendedGraphemeClusterLiteral value: String) {
        self.init(stringLiteral: value)
    }
    
    public init(unicodeScalarLiteral value: String) {
        self.init(stringLiteral: value)
    }
}
```

不过你会收到这样的报错：

> Initializer requirement 'init(stringLiteral:)' can only be satisfied by a `required` initializer in the definition of non-final class 'UIColor'

Xcode 的报错有时候不爱说人话，其实这句话的意思是说，'UIColor' 不是一个标记为 `final` 的类，也就是它还可以被继承。因此 `init(stringLiteral:)` 函数需要被标记为 `required` 以确保所有子类都实现了这个函数。否则，如果有子类没有实现它，那么子类就不满足 `StringLiteralConvertible ` 协议。

好吧，我们听从 Xcode 的指示，把每个函数都标记为 `required`，新的问题又出现了：

> 'required' initializer must be declared directly in class 'UIColor' (not in an extension)

这是因为 Swift 不允许在类型拓展中声明 `required` 函数。`required` 函数必须被直接声明在类的内部。

这就导致了一个死循环，因此目前理想方案无法实现，除非未来 Swift 允许在拓展中声明`required` 函数。

# 继承

方案二采用了变通的解决方案。首先创建一个 `UIColor` 的子类，我们可以让这个子类实现 `StringLiteralConvertible ` 协议，然后将子类对象赋值给父类：

```swift
class KtColor: UIColor, StringLiteralConvertible {
    required init(stringLiteral value: String) {
        //这里的数字是随便写的，实际上需要解析字符串
        super.init(red: 0.5, green: 0.8, blue: 0.25, alpha: 1)
    }

    required convenience init(extendedGraphemeClusterLiteral value: String) {
        self.init(stringLiteral: value)
    }

    required convenience init(unicodeScalarLiteral value: String) {
        self.init(stringLiteral: value)
    }

    required init?(coder aDecoder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }

    required convenience init(colorLiteralRed red: Float, green: Float, blue: Float, alpha: Float) {
        self.init(colorLiteralRed: red, green: green, blue: blue, alpha: alpha)
    }
}

override func viewDidLoad() {
    super.viewDidLoad()

    self.view.backgroundColor = "224, 222, 255" as KtColor
}
```

这种方法有一个显而易见的好处，一旦 Swift 做出了修改，比如允许在拓展中声明`required` 函数，我们只需要微小的改动就可以实现理想方案。

# 局限性

继承 UIKit 中的类并不总是一种可行的方法。比如 `NSDate` 类其实是一个类簇，官网对它有如下解释：

> The major reason for subclassing NSDate is to create a class with convenience methods for working with a particular calendrical system. But you could also require a custom NSDate class for other reasons, such as to get a date and time value that provides a finer temporal granularity. If you want to subclass NSDate to obtain behavior different than that provided by the private or public subclasses, you must do these things:
> 
> 列出了一大串你根本不想去做的事，省略一万字。。。。。。

简单来说，如果你想继承 `NSDate`，就必须重新实现它。

除了类簇，像 `NSURL` 这样，指定构造函数是可失败构造函数的类也无法使用继承：

```swift
class KtURL : NSURL, StringLiteralConvertible {
    required init(stringLiteral value: StringLiteralType) {
        super.init(string: value, relativeToURL: nil)
    }
    // 其他的函数略
}
```

`StringLiteralConvertible`协议中定义的构造函数是不可失败构造函数，它不会返回 `nil`。在它的内部调用了父类，也就是 `NSURL` 的 `init(string:relativetoURL:)`，这是可失败构造函数。Swift不允许出现这种情况，否则如果传入的参数 `value` 不合法，你会得到 `nil`么，如果不是 `nil` 那么会得到什么？

# 总结

完全使用字符串字面量创建已有类的实例变量在目前是无法实现的，一种想法类似但是存在局限性的方法是使用子类。或者也可以拓展 `String`类型，但如果相比于原生实现，不能较大幅度的减少代码量，我不建议这么做。

参考资料：

1. [https://devforums.apple.com/message/1057368#1057368](https://devforums.apple.com/message/1057368#1057368)：这个好像是喵神提的问题。

2. [Swift: Simple, Safe, Inflexible](https://medium.com/bloc-posts/swift-simple-safe-inflexible-68ff6fa927dc#.5ymodxskm)