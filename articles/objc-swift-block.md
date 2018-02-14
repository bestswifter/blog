最近在看Swift闭包截获变量时遇到了各种问题，总结之后发现主要是还用停留在OC时代的思维来思考Swift问题导致的。借此机会首先复习一下OC中关于block的细节，同时整理Swift中闭包的相关的问题。不管是目前使用OC还是Swift，又或者是从OC转向Swift，都可以阅读这篇文章并与我交流。

# OC 的 block

OC的block已经有很多相关的文章介绍了，主要难点在于`__block`修饰符的作用和原理，以及循环引用问题。我们首先由浅入深举几个例子看一看`__block`修饰符，最后分析循环引用问题。这里的讨论都是基于ARC的。

### 截获基本类型

```objc
int value = 10;
void(^block)() = ^{
  NSLog(@"value = %d", value);
};
value = 20;
block();

// 打印结果是："value = 10"
```

OC 的 block 会截获外部变量，对于`int`等基本数据类型，block的内部会拷贝一份，简单来说，它的实现大概是这样的：

```objc
struct block_impl {
  //其它内容
  int value;
};
```

因为block内部拷贝了截获的变量的副本，所以生成block后再修改变量，不会影响被block截获的变量。同时block内部也不能修改这个变量。

### 修改基本类型

如果要想在block中修改被截获的基本类型变量，我们需要把它标记为`__block`：

```objc
__block int value = 10;
void(^block)() = ^{
  NSLog(@"value = %d", value);
};
value = 20;
block();

// 打印结果是："value = 20"
```

这是因为，对于被标记了`__block`的变量，block在截获它时，会保存一个指针。简单来说，它的实现大概是这样的：

```objc
struct block_impl {
  //其它内容
  block_ref_value *value;
};

struct block_ref_value {
  int value; // 这里保存的才是被截获的value的值。
};
```

由于 block 中一直有一个指针指向 value，所以 block 内部对它的修改，可以影响到 block 外部的变量。因为 block 修改的就是那个外部变量而不是外部变量的副本。

上面关于block具体实现的例子只是一个简化模型，事实上并非如此，但本质类似。总的来说，只有由`__block`修饰符修饰的变量，在被block截获时才是可变的。关于这方面的详细解释，可以参考这三篇文章：

* [iOS OC语言: Block底层实现原理](http://www.jianshu.com/p/e23078c11518)：这个很详细地讲了`__block`的实现原理
* [Block的引用循环问题 (ARC & non-ARC)](http://blog.csdn.net/wildfireli/article/details/22063001)：这个讲了一些block底层的实现原理以及循环引用问题。
* [你真的理解__block修饰符的原理么？](http://blog.csdn.net/abc649395594/article/details/47086751)：这是我之前写过的一篇介绍`__block`原理的文章，内容会详细一些。

### 截获指针

block截获指针和截获基本类型是相似的，不过稍稍复杂一些。先看一个最简单的例子。

```objc
Person *p = [[Person alloc] initWithName:@"zxy"];
void(^block)() = ^{
  NSLog(@"person name = %@", p.name);
};

p.name = @"new name";
block();

// 打印结果是："person name = new name"
```

在截获基本类型时，block内部可能会有`int capturedValue = value;`这样的代码，类比到指针也是一样的，block内部也会有这样的代码：`Person *capturedP = p;`。在ARC下，这其实是强引用(retain)了block外部的`p`。

由于block内部的`p`和外部的`p`指向的是同一块内存地址。所以在block外部修改`p`的属性，依然会影响到block内部截获的`p`。

需要强调一点，这里的`p`依然不是可变的。修改`p`的`name`不是改变`p`，只是改变`p`内部的属性：

```objc
Person *p = [[Person alloc] initWithName:@"zxy"];
void(^block)() = ^{
  p.name = @"new name"; //OK，没有改变p
  p = [[Person alloc] initWithName:@"new name"]; //编译错误
  NSLog(@"person name = %@", p.name);
};

block();
```

### 改变指针

类比`__block`修饰符对基本类型的作用原理，由它修饰的指针，在被block截获时，截获的其实是这个指针的指针。比如我们把刚刚的例子修改一下：

```objc
__block Person *p = [[Person alloc] initWithName:@"zxy"];
void(^block)() = ^{
  NSLog(@"person name = %@", p.name);
};

p = nil;
block();

// 打印结果是："person name = (null)"
```

此时，block内部有一个指向外部的`p`的指针,一旦`p`被设为`nil`，这个内部的指针就指向了`nil`。所以打印结果就是`null`了。

### __block与强引用

还记得以前有一次面试时被问到，`__block`会不会`retain`变量？答案是：会的。从原理上分析，`__block`修饰的变量被封装在结构体中，block内部持有对这个结构体的强引用。这一点不管是对于基本类型还是指针都是通用的。从实际例子上来说：

```objc
Block block;
if (true) {
  __block Person *p = [[Person alloc] initWithName:@"zxy"];
  block = ^{
    NSLog(@"person name = %@", p.name);
  };
}
block();

// 打印结果是："person name = zxy"
```

如果没有`retain`被标记为`__block`的指针`p`，那么超出作用于后应该会得到`nil`。

### 避免循环引用

不管对象是否标记为`__block`,一旦block截获了它，就会强引用它。所以，判断是否发生循环引用，只要判断block截获的对象，是否也持有block即可。如果这个对象确实需要直接或间接持有block，那么我们需要避免block强引用这个对象。解决办法是使用`__weak`修饰符。

```objc
// block是self的一个属性

id __weak weakSelf = self;
block = ^{
  //使用weakSelf代替self
};
```

block不会强引用被标记为`__weak`的对象，只会对其产生弱引用。为了防止在block内的操作会释放`wself`,可以先强引用它。这种做法有一个很漂亮的名字叫`weak-strong dacne`，具体实现方法可以参考RAC的`@strongify`和`@weakify`。

### OC中block总结

简单来说，除非标记为`__weak`，block总是会强引用任何捕获的对象。而`__block`表示捕获的就是指针本身,而非另一个指向这个对象的指针。也就是说，被`__block`修饰的对象在block内、外的改动会互相影响。

如果想避免循环引用问题，首先要确定block引用了哪些对象，然后判断这些对象是否直接或间接持有block，如果有的话把这些对象标记为`__weak`避免block强引用它。

# Swift的闭包

OC中的`__block`是一个很讨厌的修饰符。它不仅不容易理解，而且在ARC和非ARC的表现截然不同。`__block`修饰符本质上是通过截获变量的指针来达到在闭包内修改被截获的变量的目的。

在Swift中，这叫做截获变量的引用。闭包默认会截取变量的引用，也就是说所有变量默认情况下都是加了`__block`修饰符的。

```swift
var x = 42
let f = {
  // [x] in //如果取消注释，结果是42
  print(x)
}
x = 43
f() // 结果是43
```

如果如果被截获的变量是引用，和OC一样，那么在闭包内部有一个引用的引用：

```swift
var block2: (() -> ())?
if true {
  var a: A? = A()
  block2 = {
    print(a?.name)
  }
  a = A(name: "new name")
}
block2?() //结果是："Optional("new name")"
```

如果把变量写在截获列表中，那么block内部会有一个指向对象的强引用，这和在OC中什么都不写的效果是一样的：

```swift
var block2: (() -> ())?
if true {
  var a: A? = A()
  block2 = { [a] in
    print(a?.name)
  }
  a = A(name: "new name")
}
block2?() //结果是："Optional("old name")"
```

Swift会自动持有被截获的变量的引用，这样就可以在block内部直接修改变量。不过在一些特殊情况下，Swift会做一些优化。通过之前OC中对`__block`的分析可以看到，持有变量的引用肯定比直接持有变量开销更大。所以Swift会自动判断你是否在闭包中或闭包外改变了变量。如果没有改变，闭包会直接持有变量，即使你没有显式的把它卸载捕获列表中。下面这句话截取自[Swift官方文档](https://developer.apple.com/library/ios/documentation/Swift/Conceptual/Swift_Programming_Language/Closures.html#//apple_ref/doc/uid/TP40014097-CH11-ID94)：

> As an optimization, Swift may instead capture and store a copy of a value if that value is not mutated by or outside a closure.

### Swift循环引用

不管是否显示的把变量写进捕获列表，闭包都会对对象有强引用。如果闭包是某个对象的属性，而且闭包中截获了对象本身，或对象的某个属性，就会导致循环引用。这和OC中是完全一样的。解决方法是在捕获列表中把被截获的变量标记为`weak`或`unowned`。

关于Swift的循环引用，有一个需要注意的例子：

```swift
class A {
  var name: String = "A"
  var block: (() -> ())?

  //其他方法
}

var a: A? = A()
var block = {
  print(a?.name)
}
a?.block = block
a = nil
block()
```

我们先创建了可选类型的变量`a`，然后创建一个闭包变量，并把它赋值给`a`的`block`属性。这个闭包内部又会截获`a`，那这样是否会导致循环引用呢？

答案是否定的。虽然从表面上看，对象的闭包属性截获了对象本身。但是如果你运行上面这段代码，你会发现对象的`deinit`方法确实被调用了，打印结果不是“A”而是“nil”。

这是因为我们忽略了可选类型这个因素。这里的`a`不是A类型的对象，而是一个可选类型变量，其内部封装了A的实例对象。闭包截获的是可选类型变量`a`，当你执行`a = nil`时，并不是释放了变量`a`，而是释放了`a`中包含的A类型实例对象。所以A的`deinit`方法会执行，当你调用block时，由于使用了可选链，就会得到`nil`，如果使用强制解封，程序就会崩溃。

如果想要人为造成循环引用，代码要这样写：

```swift
var block: (() -> ())?
if true {
  var a = A()
  block = {
    print(a.name)
  }
  a.name = "New Name"
}
block!()
```

### Weak-Strong Dance

为了避免`weak`变量在闭包中提前被释放，我们需要在block一开始强引用它。这在OC部分已经讲过如何使用了。Swift中实现Weak-Strong Dance一般有三种方法。分别是最简单的`if let`可选绑定、标准库的`withExtendedLifetime `方法和自定义的`withExtendedLifetime `方法。

# 总结

1. OC中默认截获变量，Swift默认截获变量的引用。它们都会强引用被截获的变量。
2. Swift中没有`__block`修饰符，但是多了截获列表。通过把截获的变量标记为`weak`避免引用循环
3. 两者都有Weak-Strong Dance，不过这一点上OC的写法更简单。
4. 在使用可选类型时，要明确闭包截获了可选类型还是实例变量。这样才能正确判断是否发生循环引用。
