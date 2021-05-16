# Swift 汇编（一）Protocol Witness Table 初探

由于工作中接触到 Swift 汇编与逆向知识，所以整理了这篇博客。内容与顺序无关，第一篇文章并非入门，单纯只是第一篇文章。建议有一定汇编基础的读者学习。

> 编译环境：
> MacOS 11.3.1 x86_64
> Swift: version 5.4
> 汇编风格：intel

## 什么是 Protocol Witness Table

我们知道 C 函数调用是静态派发，简单来说可以理解为是用汇编命令 `call $address` 来实现。这种方式效率最高，但是灵活性不够。

OC 的方法调用完全是基于动态派发，总是调用 `objc_msgSend` 实现。这种方式非常灵活，允许各种 Hook 黑科技，但是流程最长，效率最低。

在 Swift 中，协议方法的调用，使用协议方法表的方式完成，也就是 Protocol Witness Table，下文简称 PWT。参考下面这段代码：

```swift
protocol Drawable {
    func draw() -> Int
}

struct Point: Drawable {
    var x, y: Int
    func draw() -> Int {
        return x + y
    }
}

struct Line: Drawable {
    var length: Int
    func draw() -> Int {
        return length
    }
}

func foo() -> Int {
    let p: Drawable = xxx
    return p.draw()
}
```

在 foo 函数中，变量 p 并没有明确的类型，只知道它遵守 `Drawable` 协议，实现了 `draw` 方法。但是编译时并不能知道，调用的是结构体 `Line` 还是 `Point` 的 `draw` 方法。

因此，PWT 的实现方式是：每个类都会有一个方法表（通过数组来实现），里面保存了它用于实现协议的函数的地址。只要知道一个类的信息和函数信息，就可以实现函数调用。这个方法表，就是 PWT。

## PWT 的汇编实现

除了从理论上了解 PWT 的概念，我们还可以从汇编角度来实际感受一下。参考下面这段代码:

```swift
protocol Drawable {
    func draw() -> Int
}

struct Point: Drawable {
    var x, y: Int
    func draw() -> Int {
        return x + y
    }
}

func foo() -> Int {
    let p: Drawable = Point(x: 1, y: 2)
    return p.draw()
}
```

Debug 模式下的汇编代码：

```assembly
swift-ui-test`foo():
    0x10518a860 <+0>:   push   rbp
    0x10518a861 <+1>:   mov    rbp, rsp
    0x10518a864 <+4>:   push   r13
    0x10518a866 <+6>:   sub    rsp, 0x48
    0x10518a86a <+10>:  mov    edi, 0x1
    0x10518a86f <+15>:  mov    esi, 0x2
    0x10518a874 <+20>:  call   0x10518ae50               ; swift_ui_test.Point.init(x: Swift.Int, y: Swift.Int) -> swift_ui_test.Point at ContentView.swift:27
    0x10518a879 <+25>:  lea    rcx, [rip + 0x1948]       ; type metadata for swift_ui_test.Point
    0x10518a880 <+32>:  mov    qword ptr [rbp - 0x18], rcx
    0x10518a884 <+36>:  lea    rcx, [rip + 0x189d]       ; protocol witness table for swift_ui_test.Point : swift_ui_test.Drawable in swift_ui_test
    0x10518a88b <+43>:  mov    qword ptr [rbp - 0x10], rcx
    0x10518a88f <+47>:  mov    qword ptr [rbp - 0x30], rax
    0x10518a893 <+51>:  mov    qword ptr [rbp - 0x28], rdx
    0x10518a897 <+55>:  mov    rax, qword ptr [rbp - 0x18]
    0x10518a89b <+59>:  mov    rcx, qword ptr [rbp - 0x10]
    0x10518a89f <+63>:  lea    rdx, [rbp - 0x30]
    0x10518a8a3 <+67>:  mov    rdi, rdx
    0x10518a8a6 <+70>:  mov    rsi, rax
    0x10518a8a9 <+73>:  mov    qword ptr [rbp - 0x38], rax
    0x10518a8ad <+77>:  mov    qword ptr [rbp - 0x40], rcx
    0x10518a8b1 <+81>:  mov    qword ptr [rbp - 0x48], rdx
    0x10518a8b5 <+85>:  call   0x10518ae60               ; __swift_project_boxed_opaque_existential_1 at <compiler-generated>
    0x10518a8ba <+90>:  mov    rcx, qword ptr [rbp - 0x40]
    0x10518a8be <+94>:  mov    rdx, qword ptr [rcx + 0x8]
    0x10518a8c2 <+98>:  mov    r13, rax
    0x10518a8c5 <+101>: mov    rdi, qword ptr [rbp - 0x38]
    0x10518a8c9 <+105>: mov    rsi, rcx
    0x10518a8cc <+108>: call   rdx
->  0x10518a8ce <+110>: mov    rdi, qword ptr [rbp - 0x48]
    0x10518a8d2 <+114>: mov    qword ptr [rbp - 0x50], rax
    0x10518a8d6 <+118>: call   0x10518aec0               ; __swift_destroy_boxed_opaque_existential_1 at <compiler-generated>
    0x10518a8db <+123>: mov    rax, qword ptr [rbp - 0x50]
    0x10518a8df <+127>: add    rsp, 0x48
    0x10518a8e3 <+131>: pop    r13
    0x10518a8e5 <+133>: pop    rbp
    0x10518a8e6 <+134>: ret    
```

首先按照函数调用来分割下，这里实现了结构体的初始化工作：

```Assembly
0x10518a86a <+10>:  mov    edi, 0x1
0x10518a86f <+15>:  mov    esi, 0x2
0x10518a874 <+20>:  call   0x10518ae50               ; swift_ui_test.Point.init(x: Swift.Int, y: Swift.Int) -> swift_ui_test.Point at ContentView.swift:27
```

根据结构体的调用惯例，可以知道返回值是通过 `rax` 和 `rdx` 两个寄存器返回的。当然也可以看下这个函数的内部实现来验证下。顺便也能看出，Debug 模式下对于理解汇编代码和进行反汇编都是非常友好的，非常耿直的用一个函数调用告诉我们这里实在创建结构体实例。如果是 Release 模式，大概率是直接对 `rax` 和 `rdx` 赋值了。

接下来分别把 `metadata` 和 `Point` 类的 PWT 表取出，存到栈上。注意到下一个 call 的函数是 `__swift_project_boxed_opaque_existential_1 at <compiler-generated>`，它的存在是由于我们的这种写法导致：

```swift
let p: Drawable = Point(x: 1, y: 2)
```

这里的 p 就是一个 existential 对象，`Drawble` 协议是一个 existential type，具体的解释可以参考[这篇文章](https://stackoverflow.com/a/59183168)。简答说结论，这个函数调用以后，入参寄存器 `rdi` 的内容会被赋值给 `rax` 寄存器来当做返回值。

注意到这个函数的入参 `rdi` 寄存器，是由下面几个关键路径构成的 

```Assembly
0x10518a89f <+63>:  lea    rdx, [rbp - 0x30]
0x10518a8a3 <+67>:  mov    rdi, rdx
```

所以返回值 `rax` ，其实就是栈基址 `rbp` 减掉 `0x30`，这个地址内存贮的值，是结构体的第一个成员变量 `x = 1`。顺便说一下，这个地址向上（高地址方向）偏移 8 字节，存储的是第二个成员变量 `y = 2`。

下一个关键操作是 `call rdx`，它的取值来源是:

```Assembly
0x1073be8be <+94>:  mov    rdx, qword ptr [rcx + 0x8]
```

这里的 `rcx` 经过几次存储、取出，可以跟踪到它最初的源头，就是：

```Assembly
0x1073be884 <+36>:  lea    rcx, [rip + 0x189d]       ; protocol witness table for swift_ui_test.Point : swift_ui_test.Drawable in swift_ui_test
```

从逻辑上看，调用了 `PWT` 内存地址 + 0x8 位置的函数。可以具体看下这里到底存了何方神圣：

![](https://images.bestswifter.com/mweb/16211625755898.jpg)


* 首先看下 [rip + 0x189d] 的值是多少。在执行这行命令时，`rip` 的值是下一行命令的地址，即 `0x1073be88b`，相加后得到 `0x000000010518c128`
* 由于 Hopper、MachoView 等工具只能显示相对便宜，因此要先减去当前程序在内存中的偏移。可以用 `image list swift-ui-test` 来查看
* 得到结果是 `0x4128`


所以 `0x4128` 就是 `Point` 结构体的 PWT 的位置，可以在 Hopper 中验证下：

![](https://images.bestswifter.com/mweb/16211626037785.jpg)

这里其实是一个指针数组，第一个指针是 `0x100003998`，内容如下，暂时没有深入研究其中存储内容的含义，但是可以看出名字是：`protocol conformance descriptor for swift_ui_test.Point : swift_ui_test.Drawable in swift_ui_test`

![](https://images.bestswifter.com/mweb/16211626795756.jpg)

第二个指针是 `0x100002ff0`，跳转过去看下：

![](https://images.bestswifter.com/mweb/16211627465248.jpg)

从 demangle 后的结果也能看出来，这是一个遵守了协议的证明（Protocol Witness），遵守的协议函数是：`Drawable.draw() -> Swift.Int`，结构体是 `Point`，协议名是 `Drawable`

因此 `call rdx` 实际上就是调用 `call 0x100002ff0`

再来对比下入参和参数，`rax` 被作为 `r13` 传入了。函数内部分别把 `r13` 和 `r13 + 8` 的位置读出来，放入 `rdi` 和 `rsi` 寄存器。正如前文所述，`r13/rax` 这个地址上，存储的是 `x` 的值，`+0x8` 则存储了 `y` 的值。因此可以理解为把结构体 `p` 传入了。

最后调用了 `$s13swift_ui_test5PointV4drawSiyF` 这个函数符号，内部逻辑有点啰嗦，猜测是 Debug 环境导致，但本质上就是一个加法运算。

## 结论

至此 PWT 的调用链路就分析结束了。可以得到如下结论：

* PWT 是为了解决协议方法调用在编译时无法确定地址，而引入的中间层
* 每个遵守了协议的类，都会有自己的 PWT。遵守的协议中函数越多，PWT 中存储的函数地址就越多。
* 准确来说，PWT 是指针数组，但是第一个指针并不是函数指针，而是 `protocol conformance descriptor`，从第二个开始才是函数指针。如果有读者知道这个 `conformance descriptor` 中存储信息的含义，欢迎指教
* 对协议方法的调用，首先会调用一个 `PWT address + offset` 这个函数，这个函数被叫做 `protocol witness`，它的内部会做一些参数处理，最后再调用真实的函数
* 对于实际被调用的来说，只看它的内部实现，无法和其它函数做出区分。但是可以观察它的 caller，如果是一个 `protocol witness` 就可以说明。