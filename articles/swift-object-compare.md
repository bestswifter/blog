今天突然想到一个问题，让我觉得有必要总结一下switch语句。我们知道swift中的switch，远比C语言只能比较整数强大得多，但问题来了，哪些类型可以放到switch中比较呢，对象可以比较么？

[官方文档](https://developer.apple.com/library/ios/documentation/Swift/Conceptual/Swift_Programming_Language/ControlFlow.html#//apple_ref/doc/uid/TP40014097-CH9-ID120)对switch的用法给出了这样的解释：

> Cases can match many different patterns, including interval matches, tuples, and casts to a specific type.

也就是说除了最常用的比较整数、字符串等等之外，switch还可以用来匹配范围、元组，转化成某个特定类型等等。但文档里这个**including**用的实在是无语，因为它没有指明所有可以放在switch中比较的类型，文章开头提出的问题依然没有答案。

我们不妨动手试一下，用switch匹配对象：

```swift
class A {
    
}

var o  = A()
var o1 = A()
var o2 = A()

switch o {
case o1:
    print("it is o1")
case o2:
    print("it is o2")
default:
    print("not o1 or o2")
}
```

果然，编译器报错了：“Expression pattern of type 'A' cannot match values of type 'A'”。至少我们目前还不明白“expression pattern”是什么，怎么类型A就不能匹配类型A了。

我们做一下改动，在`case`语句后面加上`let`：

```swift
switch o {
case let o1:
    print("it is o1")
case let o2:
    print("it is o2")
default:
    print("not o1 or o2")
}
```

OK，编译运行，结果是：`it is o1`。这是因为`case let`不是匹配值，而是值绑定，也就是把o的值赋给临时变量o1,这在o是可选类型时很有用，类似于`if let`那样的隐式解析可选类型。没有打出`it is o2`是因为swift中的switch，只匹配第一个相符的case，然后就结束了，即使不写`break`也不会跳到后面的case。

扯远了，回到话题上来，既然添加`let`不行，我们得想别的办法。这时候不妨考虑一下`switch`语句是怎么实现的。据我个人猜测，估计类似于用了好多个if判断有没有匹配的case，那既然如此，我们给类型A重载一下`==`运算符试试：

```swift
class A {}

func == (lhs: A, rhs: A) -> Bool { return true }

var o = A(); var o1 = A() ;var o2 = A()

switch o {
case o1:
    print("it is o1")
case o2:
    print("it is o2")
default:
    print("not o1 or o2")
}
```

很显然，又失败了。如果这就能搞定问题，那这篇文章也太水了。报错信息和之前一样。可问题是我们已经重载了`==`运算符，为什么A类型还是不能饿匹配A类型呢，难道switch不用判断两个变量是否相等么。

switch作为一个多条件匹配的语句，自然是要判断变量是否相等的，不过它不是通过`==`运算符判断，而是通过`~=`运算符。再来看一段[官方文档](https://developer.apple.com/library/ios/documentation/Swift/Conceptual/Swift_Programming_Language/Patterns.html#//apple_ref/doc/uid/TP40014097-CH36-ID419)的解释：

> An expression pattern represents the value of an expression. Expression patterns appear only in switch statement case labels.

以及这句话：

> The expression represented by the expression pattern is compared with the value of an input expression using the Swift standard library ~= operator. 

第一句解释了之前的报错，所谓的“express pattern”是指表达式的值，这个概念只在switch的case标签中有。所以之前的报错信息是说：“o1这个表达式的值(还是o1)与传入的参数o都是类型A的，但它们无法匹配”。至于为什么不能匹配，答案在第二句话中，因为o1和o的匹配是通过调用标准库中的`~=`运算符完成的。

所以，只要把重载`==`换成重载`~=`就可以了。改动一个字符，别的都不用改，然后程序就可以运行了。Swift默认在`~=`运算符中调用`==`运算符，这也就是为什么我们感觉不到匹配整数类型需要什么额外处理。但对于自定义类型来说，不重载`~=`运算符，就算你重载了`==`也是没用的。

除此以外，还有一种解决方法，那就是让A类型实现`Equatable`协议。这样就不需要重载`~=`运算符了。答案就在Swift的module的最后几行：

```swift
@warn_unused_result
public func ~=<T : Equatable>(a: T, b: T) -> Bool
```

Swift已经为所有实现了`Equatable `协议的类重载了`~=`运算符。虽然实现`Equatable `协议只要求重载`==`运算符，但如果你不显式的注明遵守了`Equatable `协议，swift是无法知道的。因此，如果你重载了`==`运算符，就顺手标注一下实现了`Equatable `协议吧，这样还有很多好处，比如`SequenceType`的`split`方法等。

最后总结一句：
> 能放在switch语句中的类型必须重载`~=`运算符，或者实现`Equatable`协议。