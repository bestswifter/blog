本文会详细介绍一些Swift中不为大多数人知，又很有用的知识点。您不必一次性看完，不过或许哪一天这些知识就能派上用场，项目Demo在[我的github](https://github.com/bestswifter/MySampleCode/tree/master/SwiftMysterious)，您可以下载下来亲自实验一番，如果觉得有用还望点个star以示支持。

本文主要的知识点有：

* @noescape和@autoclosure
* 内联lazy属性
* 函数柯里化
* 可变参数
* dynamic关键字
* 一些特殊的字面量
* 循环标签

## @noescape和@autoclosure

关于这两个关键字的含义，在我此前的文章——[第六章——函数(自动闭包和内存)](http://www.jianshu.com/p/f9ba4c41d9c7)中已经有详细的解释，这里就简单总结概括一下：

* @noescape：这个关键字告诉编译器，参数闭包只能在函数内部使用。它不能被赋值给临时变量，不能异步调用，也不能作为未标记为@noescape的参数传递给其他函数。总之您可以放心，它无法在这个函数作用域之外使用。

除了安全性上的保证，swift还会为标记为@noescape的参数做一些优化，闭包内访问类的成员时您还可以省去`self.`的语法。

* @autoclosure：这个关键字将表达式封装成闭包，优点在于延迟了表达式的执行，缺点是如果滥用会导致代码可读性降低。

## 内联lazy属性

标记为lazy的属性在对象初始化时不会被创建，它直到第一次被访问时才会创建，通常情况下它是这样实现的：

```swift
class PersonOld {
  lazy var expensiveObject: ExpensiveObject = {
  return self.createExpensiveObject()    // 传统实现方式
}()

  private func createExpensiveObject() -> ExpensiveObject {
    return ExpensiveObject()
  }
}
```

lazy属性本质上是一个闭包，闭包中的表达式只会调用一次。需要强调的是，虽然这个闭包中捕获了`self`，但是这样做并不会导致循环引用，猜测是Swift自动把`self`标记为unowned了。

这样的写法其实可以进行简化，简化后的实现如下：

```swift
class Person {
  lazy var expensiveObject: ExpensiveObject = self.createExpensiveObject()

  private func createExpensiveObject() -> ExpensiveObject {
    return ExpensiveObject()
  }
}
```

## 函数柯里化

函数柯里化也是一个老生常谈的问题了，我的这篇文章——[第六章——函数（函数的便捷性）](http://www.jianshu.com/p/b2d21b85a387)对其有比较详细的解释。

简单来说，柯里化函数处理一个参数，然后返回一个函数处理剩下来的所有参数。直观上来看，它避免了很多括号的嵌套，提高了代码的简洁性和可读性，比如这个函数：

```swift
func fourChainedFunctions(a: Int) -> (Int -> (Int -> (Int -> Int))) {
  return { b in
    return { c in
      return { d in
        return a + b + c + d
      }
    }
  }
}
fourChainedFunctions(1)(2)(3)(4)
```

对比一下它的柯里化版本：

```swift
func fourChainedFunctions(a: Int)(b: Int)(c: Int)(d: Int) -> Int {
  return a + b + c + d
}
```

不过在 Swift 3.0 中，这种柯里化语法会被移除，你需要使用之前完整的函数声明。感谢 [@没故事的卓同学](http://www.jianshu.com/users/88a056103c02/latest_articles) 指出。

您可以在[Swift Programming Language Evolution](https://github.com/apple/swift-evolution)中查看更多细节：

![](http://images.bestswifter.com/20160129/evolution.png)

或者您也可以点击这篇文章查看更多细节——[Removing currying func declaration syntax](https://github.com/apple/swift-evolution/blob/master/proposals/0002-remove-currying.md)

## 可变参数

如果在参数类型后面加上三个"."，表示参数的数量是可变的，如果您有过Java编程的经验，对此应该会比较熟悉：

```swift
func printEverythingWithAKrakenEmojiInBetween(objectsToPrint: Any...) {
  for object in objectsToPrint {
    print("\(object)🐙")
  }
}
printEverythingWithAKrakenEmojiInBetween("Hey", "Look", "At", "Me", "!")
```

此时，参数可以当做`SequenceType`类型来使用，也就是说可以使用`for in`语法遍历其中的每一个参数。

可变参数并不是什么罕见的语法，比如`print`函数就是用了可变参数，更多详细的分析请移步：[你其实真的不懂print("Hello,world")](http://www.jianshu.com/p/abb55919c453)


## dynamic关键字

如果您有过OC的开发经验，那一定会对OC中@dynamic关键字比较熟悉，它告诉编译器不要为属性合成getter和setter方法。

Swift中也有dynamic关键字，它可以用于修饰变量或函数，它的意思也与OC完全不同。它告诉编译器使用动态分发而不是静态分发。OC区别于其他语言的一个特点在于它的动态性，任何方法调用实际上都是消息分发，而Swift则尽可能做到静态分发。

因此，标记为dynamic的变量/函数会隐式的加上@objc关键字，它会使用OC的runtime机制。

虽然静态分发在效率上可能更好，不过一些app分析统计的库需要依赖动态分发的特性，动态的添加一些统计代码，这一点在Swift的静态分发机制下很难完成。这种情况下，虽然使用dynamic关键字会牺牲因为使用静态分发而获得的一些性能优化，但也依然是值得的。

```swift
class Kraken {
  dynamic var imADynamicallyDispatchedString: String

  dynamic func imADynamicallyDispatchedFunction() {
    //Hooray for dynamic dispatch!
  }
}
```

使用动态分发，您可以更好的与OC中runtime的一些特性（如CoreData，KVC/KVO）进行交互，不过如果您不能确定变量或函数会被动态的修改、添加或使用了Method-Swizzle，那么就不应该使用dynamic关键字，否则有可能程序崩溃。

## 特殊的字面量

在开发或调试过程中如果能用好下面这四个字面量，将会起到事半功倍的效果：

* \_\_FILE__：当前代码在那个文件中
* \_\_FUNCTION__：当前代码在该文件的那个函数中
* \_\_LINE__：当前代码在该文件的第多少行
* \_\_COLUMN__：当前代码在改行的多少列

举个实际例子，您可以在demo中运行体验一番：

```swift
func specialLitertalExpression() {
  print(__FILE__)
  print(__FUNCTION__)
  print(__LINE__)
  print(__COLUMN__)   // 输出结果为11，因为有4个空格，print是五个字符，还有一个左括号。
}
```

一般情况下最常用的字面量是`__FUNCTION__ `，它可以很容易让程序员明白自己调用的方法的方法名。

## 循环标签

通常意义上的循环标签主要是`continue`和`break`，不过swift在此基础上做了一些拓展，比如下面这段代码：

```swift
let firstNames = ["Neil","Kt","Bob"]
let lastNames = ["Zhou","Zhang","Wang","Li"]
for firstName in firstNames {
var isFound = false
for lastName in lastNames {
  if firstName == "Kt" && lastName == "Zhang" {
    isFound = true
    break
  }
  print(firstName + " " + lastName)
}

  if isFound {
    break
  }
}
```

目的是希望找到分别在两个数组中找到字符串"Kt"和"Zhang"，在此之前会打印所有遍历到的字符。

在结束内层循环后，我希望外层循环也随之立刻停止，为了实现这个功能，我不得不引入了`isFound `参数。然而实际上我需要的只是可以指定停止哪个循环而已：

```swift
outsideloop: for firstName in firstNames {
  innerloop: for lastName in lastNames {
    if firstName == "Kt" && lastName == "Zhang" {
      break outsideloop	//人为指定break外层循环
    }
    print(firstName + " " + lastName)
  }
}
```

以上两段代码等价，可以看到使用了循环标签后，代码明显简洁了很多。
