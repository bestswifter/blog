在进行调试的时候，我们有时会把一个变量自身，或其成员属性的值打印出来以检查是否符合我们的预期。或者干脆简单一些，直接`print`整个变量，不同于C++的`std::cout`，如果你调用`print(value)`，不管`value`是什么类型程序都不会报错，而且大多数时候你能获得比较全面的、可读的输出结果。如果这引起了你对`print`函数的好奇，接下来我们共同研究以下几个问题：

1. `print("hello, world")`和`print(123)`的执行原理
2. `Streamable`和`OutputStreamType`协议
3. `CustomStringConvertible`和`CustomDebugStringConvertible`协议
4. 为什么字符串的初始化函数中可以传入任何类型的参数
5. `print`和`debugPrint`函数的区别

本文的demo地址在[我的github](https://github.com/bestswifter/MySampleCode/tree/master/Streamable)，读者可以下载下来自行把玩，如果觉得有收获还望给个star鼓励一下。

# 字符串输出

有笑话说每个程序员的第一行代码都是这样的：

```swift
print("Hello, world!")
```

先别急着笑，您还真不一定知道这行代码是怎么运行的。

首先，`print`函数支持重载，Swift定义了两个版本的实现。其中简化版的`print`将输出流指定为标准输出流，我们忽略playground相关的代码，来看一下上面那一行代码中的`print`函数是怎么定义的，在不改编代码逻辑的前提下，为了方便阅读，我做了一些排版方面的修改：

```swift
// 简化版print函数，通过terminator = "\n"可知print函数默认会换行
public func print(items: Any..., separator: String = " ", terminator: String = "\n") {
	var output = _Stdout()	
	_print(items, separator: separator, terminator: terminator, toStream: &output)
}

// 完整版print函数，参数中多了一个outPut参数
public func print<Target: OutputStreamType>(items: Any...,separator: String = " ",
  terminator: String = "\n",
  inout toStream output: Target) {
  _print(items, separator: separator, terminator: terminator, toStream: &output)
}
```

两者的区别在于完整版`print`函数需要我们提供`output`参数，而我们之前调用的显然是第一个`print`函数，在函数中创建了`output`变量。这两个版本的`print`函数都会调用内部的`_print`函数。

通过这一层封装，真正的核心操作在`_print`函数中，而对外则提供了一个重载的，高度可定制的`print`函数，接下来我们看一看这个内部的`_print`函数是如何实现的，为了阅读方便我删去了读写锁相关的代码，它的核心步骤如下：

```swift
internal func _print<Target: OutputStreamType>(
  items: [Any],
  separator: String = " ",
  terminator: String = "\n",
  inout toStream output: Target
) {
  var prefix = ""
  for item in items {
    output.write(prefix)	// 每两个元素之间用separator分隔开
    _print_unlocked(item, &output)	// 这句话其实是核心
    prefix = separator
  }
  output.write(terminator)	// 终止符，通常是"\n"
}
```

这个函数有四个参数，显然第一个和第四个参数是关键。也就是说我们只关心要输出什么内容，以及输出到哪里，至于输出格式则是次要的。所以`_print`函数主要是处理了输出格式问题，以及把第一个参数(它是一个数组)中的每个元素，都写入到`output`中。通过目前的分析，我们已经明白文章开头的`print("Hello, world!")`其实等价于：

```swift
var output = _Stdout()	// 这个是output的默认值
output.write("")	// prefix是一个空字符串
_print_unlocked("Hello, world!", &output)
```

你一定已经很好奇这个反复出现的`output`是什么了，其实在整个`print`函数的执行过程中`OutputStreamType `类型的`output`变量都是关键。另一个略显奇怪的点在于，同样是输出空字符串和`"Hello, world!"`，竟然调用了两个不同的方法。接下来我们首先分析`OutputStreamType `协议以及其中的`write`方法，再来研究为什么还需要`_print_unlocked`函数：

```swift
public protocol OutputStreamType {
  mutating func write(string: String)
}

internal struct _Stdout : OutputStreamType {
  mutating func write(string: String) {
    for c in string.utf8 {
      _swift_stdlib_putchar(Int32(c))
    }
  }
}
```

简单来说，`OutputStreamType`表示了一个输出流，也就说你要把字符串输出到哪里。如果你有过C++编程经验，那你一定知道`#include <iostream>`这个库文件，以及`cout`和`cin`这两个标准输出、输入流。

在`OutputStreamType `协议中定义了`write`方法，它表示这个流是如何把字符串写入的。比如标准输出流`_Stdout `的处理方法就是在字符串的UFT-8编码视图下，把每个字符转换成Int32类型，然后调用`_swift_stdlib_putchar `函数。这个函数在`LibcShims.cpp`文件中定义，可以理解为一个适配器，它内部会直接调用C语言的`putchar`函数。

Ok，已经分析到C语言的`putchar`函数了，再往下就不必说了(我也不懂`putchar`是怎么实现的)。现在我们把思路拉回到另一个把字符串打印到屏幕上的函数——`_print_unlocked `上，它的定义如下：

```swift
internal func _print_unlocked<T, TargetStream : OutputStreamType>(value: T, inout _ target: TargetStream) {
  if case let streamableObject as Streamable = value {
    streamableObject.writeTo(&target)
    return
  }

  if case let printableObject as CustomStringConvertible = value {
    printableObject.description.writeTo(&target)
    return
  }

  if case let debugPrintableObject as CustomDebugStringConvertible = value {
    debugPrintableObject.debugDescription.writeTo(&target)
    return
  }

  _adHocPrint(value, &target, isDebugPrint: false)
}
```

在调用最后的`_adHocPrint`方法之前，进行了三次判断，分别判断被输出的`value`(在我们的例子中是字符串"Hello, world!")是否实现了指定的协议，如果是，则调用该协议下的`writeTo`方法并提前返回，而最后的`_adHocPrint`方法则用于确保，任何类型都有默认的输出。稍后我会通过一个具体的例子来解释。

这里我们主要看一下`Streamable`协议，关于另外两个协议的介绍您可以参考[《第七章——字符串(字符串调试)》](http://www.jianshu.com/p/8b39e9c84462)。`Streamable`协议定义如下：

```swift
/// A source of text streaming operations.  `Streamable` instances can
/// be written to any *output stream*.
public protocol Streamable {
  func writeTo<Target : OutputStreamType>(inout target: Target)
}
```

根据官方文档的定义，`Streamable`类型的变量可以被写入任何一个输出流中。`String`类型实现了`Streamable`协议，定义如下：

```swift
extension String : Streamable {
  /// Write a textual representation of `self` into `target`.
  public func writeTo<Target : OutputStreamType>(inout target: Target) {
    target.write(self)
  }
}
```

看到这里，`print("Hello, wrold!")`的完整流程就算全部讲完了。还留下一个小疑问，同样是输出字符串，为什么不直接调用`write`函数，而是大费周章的调用`_print_unlocked `函数？这个问题在讲解完`_adHocPrint `函数的原理后您就能理解了。

需要强调一点，千万不要把`writeTo`函数和`write`函数弄混淆了。`write`函数是输出流，也就是`OutputStreamType `类型的方法，用于输出内容到屏幕上，比如`_Stdout `的`write`函数实际上会调用C语言的`putchar`函数。

`writeTo`函数是可输出类型(也就是实现了`Streamable `协议)的方法，它用于将该类型的内容输出到某个流中。

输出字符串的过程中，这两个函数的关系可以这样简单理解：

> 内容.writeTo(输出流) = 输出流.write(内容)，一般在前者内部执行后者

字符串不仅是可输出类型(Streamable)，同时自身也是输出流(OutputStreamType)，它是Swift标准库中的唯一一个输出流，定义如下：

```swift
extension String : OutputStreamType {
  public mutating func write(other: String) {
    self += other
  }
}
```

在输出字符串的过程中，我们用到的是字符串可输出的特性，至于它作为输出流的特性，会在稍后的例子中进行讲解。

# 实战

接下来我们通过几个例子来加深对`print`函数执行过程的理解。

### 一、字符串输出

还是用文章开头的例子，我们分析一下其背后的步骤：

```swift
print("Hello, world!")
```

1. 调用不带`output`参数的`print`函数，函数内部生成`_Stdout `类型的输出流，调用`_print`函数
2. 在`_print`函数中国处理完`separator`和`terminator `等格式参数后，调用`_print_unlocked `函数处理字符串输出。
3. 在`_print_unlocked `函数的第一个if判断中，因为字符串类型实现了`Streamable `协议，所以调用字符串的`writeTo`函数，写入到输出流中。
4. 根据字符串的`writeTo`函数的定义，它在内部调用了输出流的`write`方法
5. `_Stdout`在其`write`方法中，调用C语言的`putchar`函数输出字符串的每个字符

### 二、标准库中其他类型输出

如果要输出一个整数，似乎和输出字符串一样简单，但其实并不是这样，我们来分析一下具体的步骤：

```swift
print(123)
```

1. 调用不带`output`参数的`print`函数，函数内部生成`_Stdout `类型的输出流，调用`_print`函数
2. 在`_print`函数中国处理完`separator`和`terminator `等格式参数后，调用`_print_unlocked `函数处理字符串输出。
3. 截止目前和输出字符串一致，不过Int类型(以及其他除了和字符有关的几乎所有类型)没有实现`Streamable `协议，它实现的是`CustomStringConvertible `协议，定义了自己的计算属性`description`
4. `description`是一个字符串类型，调用字符串的`writeTo`方法此前已经讲过，就不再赘述了。

### 三、自定义结构体输出

我们简单的定义一个结构体，然后尝试使用`print`方法输出这个结构体：

```swift
struct Person {
    var name: String
    private var age: Int
    
    init(name: String, age: Int) {
        self.name = name
        self.age = age
    }
}

let kt = Person(name: "kt", age: 21)
print(kt)	// 输出结果：PersonStruct(name: "kt", age: 21)
```

输出结果的可读性非常好，我们来分析一下其中的步骤：

1. 调用不带`output`参数的`print`函数，函数内部生成`_Stdout `类型的输出流，调用`_print`函数
2. 在`_print`函数中国处理完`separator`和`terminator `等格式参数后，调用`_print_unlocked `函数处理字符串输出。
3. 在`_print_unlocked `中调用`_adHocPrint `函数
4. switch语句匹配，参数类型是结构体，执行对应case语句中的代码

前两步和输出字符串一模一样，不过由于是自定义的结构体，而且没有实现任何协议，所以在第三步骤无法满足任意一个if判断。于是调用`_adHocPrint `函数，这个函数可以确保任何类型都能在`print`方法中较好的工作。在`_adHocPrint `函数中也有switch判断，如果被输出的变量是一个结构体，则会执行对应的操作，代码如下：

```swift
internal func _adHocPrint<T, TargetStream : OutputStreamType>(
    value: T, inout _ target: TargetStream, isDebugPrint: Bool
) {
  func printTypeName(type: Any.Type) {
    // Print type names without qualification, unless we're debugPrint'ing.
    target.write(_typeName(type, qualified: isDebugPrint))
  }

  let mirror = _reflect(value)
  switch mirror {
  case is _TupleMirror:
    // 这里定义了输出元组类型的方法

  case is _StructMirror:
    printTypeName(mirror.valueType)
    target.write("(")
    var first = true
    for i in 0..<mirror.count {
      if first {
        first = false
      } else {
        target.write(", ")
      }
      let (label, elementMirror) = mirror[i]
      print(label, terminator: "", toStream: &target)
      target.write(": ")
      debugPrint(elementMirror.value, terminator: "", toStream: &target)
    }
    target.write(")")

  case let enumMirror as _EnumMirror:
    // 这里定义了输出枚举类型的方法

  case is _MetatypeMirror:
    // 这里定义了输出元类型的方法

  default:
    // 如果都不是就进行默认输出
  }
}
```

您可以仔细阅读`case is _StructMirror`这一段，它的逻辑和结构体的输出结果是一致的。如果此前定义的不是结构体而是类，那么得到的结果只是`Streamable.PersonStruct`，根据`default`段中的代码也很容易理解。

正是由于`_adHocPrint `方法，不仅仅是字符串和Swift内置的类型，任何自定义类型都可以被输出。现在您应该已经明白，为什么输出`prefix`用的是`write`方法，而输出字符串`"Hello, world!"`要用`_print_unlocked `函数了吧？这是因为在那个时候，编译器还无法判定输出内容的类型。

### 四、万能的String

不知道您有没有注意到一个细节，`String`类型的初始化函数是一个没有类型约束的范型函数，也就是说任意类型都可以用来创建一个字符串，这是因为`String`类型的初始化函数有一个重载为：

```swift
extension String {
  public init<T>(_ instance: T) {
    self.init()
    _print_unlocked(instance, &self)
  }
}
```

这里的字符串不是一个可输出类型，而是作为输出流来使用。`_print_unlocked `将`instance`输出到字符串流中。

# 调试输出

在`_print_unlocked`函数中，我们看到它在输出默认值之前，一共会进行三次判断。依次检验被输出的变量是否实现了`Streamable `、`CustomStringConvertible `和`CustomDebugStringConvertible `，只要实现了协议，就会进行相应的处理并提前退出函数。

这三个协议的优先级依次降低，也就是如果一个类型既实现了`Streamable `协议又实现了`CustomStringConvertible `协议，那么将会优先调用`Streamable `协议中定义的`writeTo`方法。从这个优先级顺序来看，`print`函数更倾向于字符串的正常输出而非调试输出。

Swift中还有一个`debugPrint`函数，它更倾向于输出字符串的调试信息。调用这个函数时，三个协议的优先级完全相反：

```swift
extension PersonDebug: CustomStringConvertible, CustomDebugStringConvertible {
    var description: String {
        return "In CustomStringConvertible Protocol"
    }

    var debugDescription: String {
        return "In CustomDebugStringConvertible Protocol"
    }
}

let kt = PersonDebug(name: "kt", age: 21)
print(kt)	// "In CustomStringConvertible Protocol"
debugPrint(kt)	//"In CustomDebugStringConvertible Protocol"
```

刚刚我们说到，创建字符串时可以传入任意的参数value，最后的字符串的值和调用`print(value)`的结果完全相同，这是因为两者都会调用`_print_unlocked `方法。对应到`debugPrint`函数则有：

```swift
extension String {
  public init<T>(reflecting subject: T) {
    self.init()
    debugPrint(subject, terminator: "", toStream: &self)
  }
}
```

简单来说，在`_adHocPrint `函数之前，这两个输出函数的调用栈是完全平行的关系，下面这张图作为两者的比较，也是整篇文章的总结，纯手绘，美死早：

![print与debugPring调用栈](http://images.bestswifter.com/print.png)