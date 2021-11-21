# 浅谈 swiftinterface 文件

按照惯例，首先介绍下什么是 `.swiftinterface` 文件。这个文件的作用，主要是用于描述一个 Swift 模块的公开信息。如果单从形态上来说，可以简单类比 C 系列语言的 `.h` 头文件。

但最重要的区别是，`.swiftinterface` 文件除了描述包括结构、变量、函数在内的各种公开符号以外，它还负责描述模块稳定性相关的信息，因此在这个文件内可以看到很多和内联相关的逻辑。

另外需要掌握的一个关键概念是，`.swiftinterface` 文件在编译模块时由编译器产生，并且会参与模块的消费方的编译流程。在下文中会具体解释这句话。

## 模块稳定性(`Module Stability`)

模块稳定性是一种比二进制稳定性更加要求严格的稳定性。假设目前有上游模块 A 依赖下游模块 B，二进制稳定性指的是，即使模块 A 和 B 分别由不同版本的 Swift 编译器构建，他们组合在一起要能正常工作。

而模块稳定性，则是在模块 B 的内部发生正常优化（不包括删除 API 这样的行为）时，还要满足：

1. 即使是比较古老的 A 模块，也要能和更新后的 B 模块组合起来正常工作
2. 即使 A 模块有了改动，只要还在调用当前 B 模块的公开接口，也要能正常工作

具体到技术和实现层面，模块稳定性其实可以细分为它的内部实现和对外描述两个话题来讨论。从内部实现角度来看，模块稳定性在一般情况下总是能被满足的，但是一旦涉及到内联相关的优化，就需要单独考虑了。

比如这段代码：

```swift
public struct People {
    
    internal var name: String
    
    internal var age: Int
    
    public init(name: String, age: Int) {
        self.name = name
        self.age = age
    }
    
}
```

这是一个很简单的 `People` 结构体，如果把它构建出一个单独的模块，那就就是具备模块稳定性的。但有些时候，我们会觉得它的初始化函数过于简单，没有必要发生一次函数调用， 所以可以标记为 `@inlinable`，告诉编译器可以考虑对它的初始化函数做优化。

代码如下所示：

```swift
public struct People {
    
    @usableFromInline
    internal var name: String
    
    @usableFromInline
    internal var age: Int
    
    @inlinable
    public init(name: String, age: Int) {
        self.name = name
        self.age = age
    }
    
}

```

由于 `@inlinable` 是可以跨模块的，此时的 `People` 结构体就不再具备模块稳定性。原因是：当它的上游组件在调用 `init` 函数时，可能就不是进行函数调用，而是直接在栈上分配一块内存，在 `0x0` 的偏移位置放一个字符串，然后在 `0x8` 的位置放一个整型数字。

如果后续 `People` 结构发生了变化，比如第一个成员不再是 `age`，而是添加了一个 `var id: UUID`。此时重新发布的 `People` 模块，就无法与之前编译的上游代码共存了。

但就这个 case 来看，解决方案也很简单，可以把类型标记为 `@frozen` 来承诺后续不会发生布局改动，或者不再把 `init` 函数标记为 `@inlinable`。

如果想要知道自己的代码是否具备模块稳定性，可以在 Build Settings 中找到并且打开 **`Build Libraries for Distribution`** 这个选项。打开这个选项等价于增加两个 `swiftc` 的编译参数：`-enable-library-evolution` 和 `-emit-module-interface-path`。

`-enable-library-evolution` 选项会以模块稳定的方式编译代码，将 `@inlinable` 这类函数用到的变量特殊处理，从直接的内存偏移修改为间接访问。它对应的是模块稳定性的内部实现部分。

为了更好的解释模块稳定性的原理，以及我们是如何在性能和灵活性之间进行取舍的，这里再举一个简单的例子，修改代码如下：

```swift
public struct People {
    
    internal var name: String
    
    @usableFromInline
    internal var age: Int
    
    public init(name: String, age: Int) {
        self.name = name
        self.age = age
    }
    
    @inlinable
    public func nextAge() -> Int {
        return self.age + 1
    }
    
}
```

这样的 `Poeple` 也是满足模块稳定性的，在启用 `-enable-library-evolution` 编译参数时，它的 `nextAge` 函数实现如下：

![inlinable-stability](https://images.bestswifter.com/mweb/inlinable-stability.png)

其中的 `rax` 来自于 `age.setter` 这个函数的偏移

![inlinable-age-setter](https://images.bestswifter.com/mweb/inlinable-age-setter.png)

由于 `nextAge` 函数只依赖 `age.setter` 的存在，所以后续不管 `People` 结构体的布局如何发生变化，只要这个函数还存在，都可以正确的读取到 `age` 的值。

如果不使用 `-enable-library-evolution` 参数，或者把 `People` 结构体标记为 `@frozen`，就意味着我们无需考虑结构体布局发生变动，此时的 `nextAge` 函数实现如下：

![inlinable-none-stability](https://images.bestswifter.com/mweb/inlinable-none-stability.png)

这种写法实际上依赖于小结构体可以以 `loadable` 的方式加载在寄存器中，`rdx` 即表示了 `age` 的偏移情况。一旦 `People` 的结构发生变化，`rdx` 寄存器就几乎不可能再能用于表示 `age`，这样的实现如果被内联到客户端的代码中，是万万不能保证模块稳定性的。

## interface 文件介绍

上一节说到，打开 **`Build Libraries for Distribution`** 这个选项本质上是增加两个 `swiftc` 的编译参数：`-enable-library-evolution` 和 `-emit-module-interface-path`。并且介绍了前者是如何影响编译器生成对应的函数实现的，

后一个参数的作用就是生成本文的主角： `.swiftinterface` 文件，它对应了模块稳定性的对外描述部分。其实也可以单独使用 `-emit-module-interface-path` 这个参数进行编译，虽然也可以产生 `.swiftinterface` 文件，但是会报警告：`warning: module interfaces are only supported with -enable-library-evolution`，表示 `.swiftinterface` 文件必须配合 `-enable-library-evolution` 参数使用。

`.swiftinterface` 文件是 Swift 在 5.1 版本引入的概念，它以文本的方式描述了一个模块的所有公开符号信息（包括内联相关）。在此之前，相关的信息保存在 `.swiftmodule` 文件中，这个文件是二进制格式的，不可读，且在不同版本的 Swift 编译下的实现都不一样。这也就意味着任何存在两个依赖关系二进制模块，都必须使用相同的 Swift 版本进行编译。否则依赖方就无法正确的解析被依赖方的公开符号信息。与此相比，`.swiftinterface` 文件最大的优势是，高版本的编译器总是可以兼容低版本编译器产生的 `.swiftinterface` 文件。

上一节的代码编译后产出的 `.swiftinterface` 文件如下：

```swift
// swift-interface-format-version: 1.0
// swift-compiler-version: Apple Swift version 5.5.1 (swiftlang-1300.0.31.4 clang-1300.0.29.6)
// swift-module-flags: -target x86_64-apple-ios15.0-simulator -enable-objc-interop -enable-library-evolution -swift-version 5 -enforce-exclusivity=checked -O -module-name TestLibraryEvolutoin
import Swift
import _Concurrency
@frozen public struct People {
  internal var name: Swift.String
  @usableFromInline
  internal var age: Swift.Int
  public init(name: Swift.String, age: Swift.Int)
  @inlinable public func nextAge() -> Swift.Int {
        return self.age &+ 1
    }
}
extension TestLibraryEvolutoin.People : Swift.Sendable {}
```

这里就引出了文章开头重点强调的一个概念：**`.swiftinterface` 文件在编译模块时由编译器产生，并且会参与模块的消费方的编译流程**

当我们在客户端代码中引入 `Person` 这个模块时，使用的是客户端的 Swift 编译器。此时编译器首先会检查第三行注释中有没有 `-enable-library-evolution` 编译参数，如果没有，那么编译器认为被依赖的 `Person` 模块不具备模块稳定性。此时相当于降级到了 Swift 5.1 之前的模式，会判断生成这个 `.swiftinterface` 文件的编译器版本，和消费的编译器版本是否一致。如果不一致就会报编译错误：

![error](https://images.bestswifter.com/mweb/error.png)

如果根据报错信息去搜索，得到的建议一般是让 `Person` 这个模块，先打开 `Build Libraries for Distribution` 选项，再交付具有模块稳定性的二进制产物。

但是根据笔者的经验，还有两类常见的问题，也会导致类似的错误，需要加以注意：

1. 这个 `.swiftinterface` 文件会被使用方的编译器消费，并且参与编译流程，但是这个文件本身并非总是能通过编译的。比如引用了一些不存在的模块，甚至是被人为误改。但是 Swift 编译器的处理原则是：只要 `.swiftinterface` 文件无法被正确消费，都会报上述错误。如果不清楚这个细节，就可能导致错过了真正有问题的地方。
2. 高版本的 Swift 编译器可能会用到一些低版本编译器不支持的特性。因此模块稳定性不是无条件的，它要求消费方的编译器版本不低于生产方的版本。

一个简单的例子就是这里的 `import _Concurrency`。根据测试，只要使用 Xcode13 以上进行编译（对应 Swift5.5+），编译器都会在生成的 `.swiftinterface` 文件中自动补上这句话。

## @_alwaysEmitIntoClient

`.swiftinterface` 文件除了要符合语法要求，它还会参与到编译，并且影响客户端代码的行为。一个典型的例子就是 `@_alwaysEmitIntoClient` 这个关键字的使用。

我们知道系统动态库是跟随 iOS 系统发布的，预置在操作系统中，比如最典型的 `UIKit`。笔者曾经一度以为，一旦已经发布的系统库，API 和行为就是完全确定，不可更改的。直到 iOS15 系统发布后，我突然发现了一个支持到 iOS13 的 API，而这个 API 在笔者的印象中，绝对不存在与 iOS13 上。

苹果实现这种为已发布的 Swift 二进制库打补丁的能力，就是通过 `@_alwaysEmitIntoClient` 关键字来实现的。对于被标记为 `@_alwaysEmitIntoClient` 的函数，它的函数实现也会完整的出现在 `.swiftinterface` 文件中，如果说 `@inlinable` 是告诉编译器这个函数可以被内联，那么 `@_alwaysEmitIntoClient` 就是告诉编译器，这个函数一定要被内联到客户端代码中。

假设我们的 `Person` 模块是一个系统库，对于已经发布的 `Person` 模块来说，它只有 `nextAge` 函数，并且已经预置在了所有的 iOS 系统中。此时我们可以新增一个 `previousAge` 函数，并且把它标记为 `@_alwaysEmitIntoClient`。只要新的 Xcode 携带了这份 `.swiftinterface` 文件，那么在使用新 Xcode 编译时，用户就可以使用 `previousAge` 函数，并且可以通过编译。

在实现层面，虽然已经存在的 `Person` 二进制并没有这个函数，但是在编译时，编译器看到了这个关键字，就不会真的调用 `previousAge` 函数，而是把函数的实现内联在调用方的代码中。只要这个函数的实现在低版本存在对应符号，就可以组合出任意多的新函数。

不过通过 `@_alwaysEmitIntoClient` 来打补丁，做到向前兼容，除了工程上不合理，污染 `.swiftinterface` 文件外，另一个缺点在于包大小的劣化。因为本来属于模块内的实现，只需要定义一个函数，现在会被内联分散到所有的调用方，个人并不推荐使用这种技巧。

## 总结

本文主要介绍了 `.swiftinterface` 文件和模块稳定性的背景，它的工作原理以及常见的误区，最后介绍了 `@_alwaysEmitIntoClient` 关键字以及它的实现原理。有一些简单的结论可供参考：

1. 模块稳定性主要可以分为内部实现和外部描述两个方面
   1. 内部实现主要是在内联优化与后续迭代灵活性之间进行的取舍
   2. 外部描述主要是 Swift 5.1 开始新引入的 `.swiftinterface` 文件，取代了此前版本间不兼容的 `.swiftmodule` 文件
2. 在 Xcode 中打开 `Build Libraries for Distribution` 开关，启用模块稳定性，本质是增加了两个 `swiftc` 的编译参数：`-enable-library-evolution` 和 `-emit-module-interface-path`。分别对应了内部的实现和对外描述
3. `.swiftinterface` 文件由构建方的编译器生成，但是会参与到使用方的编译流程中。不管是文件内容有语法错误，还是使用方的编译器版本低于生产方，都会报相同的编译错误，需要掌握排查技巧
4. 基于 `.swiftinterface` 文件的工作原理，苹果可以通过 `@_alwaysEmitIntoClient` 关键字并且在新的 Xcode 中更新 `.swiftinterface` 文件，从而为已经在低版本 iOS 系统上发布的系统库增加新的 API