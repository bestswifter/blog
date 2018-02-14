# Swift中的集合

最近翻译完了[《Advanced Swift》中文版](http://www.jianshu.com/p/18744b078508)的“集合”章节。书的质量非常高，讲解非常细致。但不可避免的导致篇幅有点长，有些前面的知识点看到后面无法串联起来。同时由于偏重于讲解，所以个人感觉总结还不够，比如我们可以考虑这几个问题：

* 数组类型(_ArrayType)、集合(Collection)、序列(Sequence)、生成器(Generator)、元素(Element)、下标(Index)，这些类型(协议)各自的作用。
* 数组是如何利用上面这些类实现各种方法的。
* `map`、`reduce`、`filter`等方法的作用是什么，他们是怎么实现的。
* 只有数组有上面这些方法么，如果不是，什么样的类型才有这些方法。
* 如果实现一个自定义的集合类型，应该怎么做。

而这些问题恰恰是过去我们没有重视或根本无法找到答案的问题。因为在OC中，由于`NSArray`封装的非常好，而且在单纯的iOS开发中对数组的理解不用非常深入，所以我们很少深究数组背后的实现原理。

其实答案就遍布在[《Advanced Swift》中文版](http://www.jianshu.com/p/18744b078508)的“集合”章节的各篇文章中。本文会通过分析Swift中数组的实现，回答上述问题并尝试着建立完整的知识体系。由于篇幅所限，本文不会给出非常详细的源码，而是做总结性的提炼。

# 参考文章

* [Advanced Swift——数组可变性](http://www.jianshu.com/p/21d518da46ed)：主要介绍数组的值语义
* [Advanced Swift——数组变换](http://www.jianshu.com/p/7b1bb303ec45)： 主要介绍`map`、`reduce`、`filter`等方法的使用和原理
* [Advanced Swift——字典与集合](http://www.jianshu.com/p/17731dd5db55)：主要介绍字典与集合类型的使用
* [Advanced Swift——集合协议](http://www.jianshu.com/p/17731dd5db55)：主要介绍数组相关的三种协议
* [Advanced Swift——集合](http://www.jianshu.com/p/847797ecb79b)：通过实现一个队列，介绍`CollectionType`类型的使用
* [Advanced Swift——下标](http://www.jianshu.com/p/a22e29b75e40)：实现自定义链表，介绍数组下标的相关知识

# 相关类型简介

我们从零开始，根据集合最本质的特点开始思考，逐步抽象。

### 元素(Element)

对于任何一种集合来说，它本质上是一种数据结构，也就是用来存放数据的。我们在各种代码中见到的`Element`表示的就是最底层数据的类型别名。因为对于集合这种范型类型来说，它不关心具体存放的数据的类型，所以就用通用的`Element`来代替，比如我们写这样的代码:

```swift
let array: Array<Int> = [1,2,3]
```

这就相当于告诉数组，`Element`现在具体变为`Int`类型了。

### 生成器(Generator)

对于集合来说，它往往还需要提供遍历的功能。那怎么把它设计为可遍历的呢？不要指望for循环，要知道目前我们什么都没有，没有数组的概念，更没有下标的概念。有一句名言说：“任何编程问题都可以通过增加一个中间层来解决”。于是，我们抽象出一个`Generator `(生成器)的概念。我们把它声明成一个协议，任何实现这个协议的类都要提供一个`next`方法，返回集合中的下一个元素：

```swift
protocol GeneratorType {
	typealias Element
	mutating func next() -> Element?
}
```

可以想象的是这样一来，实现了`GeneratorType `协议的类，就可以隐藏具体元素的信息，通过不断调用`next`方法就可以获得所有元素了。比如你可以用一个生成器，不断生成斐波那契数列的下一个数字。

总的来说，生成器(`Generator `)允许我们遍历所有的元素。

### 序列(Sequence)

生成器不断生成下一个元素，但是已经生成过的元素就无法再获取到了。比如在斐波那契数列中通过3和5得到了8，那么这个生成器永远也不会再生成元素3了，下一个生成的元素是13。也就是说对于生成器来说，已经访问过的元素就无法再次获取到。

然而对于集合来说，它所包含的元素总是客观存在的。为了能够多次遍历集合，我们抽象出了序列(`Sequence `)的概念。`Sequence `封装了`Generator `对象的创建过程：

```swift
protocol SequenceType {
	typealias Generator: GeneratorType
	func generate() -> Generator
}
```

序列(`Sequence `)相比于单纯的`Generator `，使**反复**遍历集合中的元素成为可能。这时候，我们已经可以通过`for`循环而不是`Generator`来遍历所有元素。

### 集合(Collection)

对比一下现有的`Sequence `和数组，会发现它还欠缺一个特性——下标。

回顾一下`Generator `和`Sequence `，它们只是实现了集合的遍历，但没有指定怎么遍历。也就是说，只要`Generator `设计“得当”，即使是1和2这两个元素，我们也可以不断遍历：“1的next是2，2的next是1”。这种情况显然不符合我们对数组的认识。归根结底，还是`Sequence `中无法确定元素的位置，也就无法确保不遍历到已经访问过的元素。

基于这种考虑，我们抽象出集合(`Collection`)的概念。在集合中，每个元素都有确切的位置，因此集合有明确的开始位置和结束位置。给定一个位置，就可以找到这个位置上的元素。`Collection`在`Sequence`的基础上实现了`Indexable`协议

```swift

public protocol CollectionType : Indexable, SequenceType {
	public var startIndex: Self.Index { get }
	public var endIndex: Self.Index { get }
	public subscript (position: Self.Index) -> Self._Element { get }
}
```

当然，`CollectionType`中的内容远远不止这些。这里列出的仅仅是为了表示`CollectionType`的特性。

### 下标(Index)

虽然我们在使用数组的时候，元素下标总是从0开始，并且逐个递增。但下标不必是从0开始递增的整数。比如`a、b、c……`也可以作为下标。下标类型的关键在于能根据某一个下标推断出下一个下标是什么，比如对于整数来说下一个下标就是当前下标值加1。下标类型的协议如下：

```swift
public protocol ForwardIndexType : _Incrementable {
	///....
}

public protocol _Incrementable : Equatable {
    public func successor() -> Self
}
```

对于下标类型来说，它们必须实现`successor()`方法，同时也得是`Equatable`的，否则怎么知道某个位置上的元素已经被访问过了呢。

# 数组简介

有了以上基本知识做铺垫，接下来就可以研究一下Swift中数组是怎么实现的了。

### 基本定义
我们可能早已体会到Swift数组的强大，它的下标脚本不仅可以读取元素，还可以直接修改元素。它可以通过字面量初始化，还可以调用
`appendContentsOf `方法从别的数组中添加元素。甚至于我们可能都没有仔细考虑过为什么可以用`for number in array`这样的语法。

首先，我们需要明确一个概念：数组丰富强大的特性绝不是由`Array`这一个类型可以提供的。实际上，这主要是协议的功劳。用OC写iOS时，协议几乎就是代理的代名词(可能是我对OC不甚精通，目光短浅)，但毋庸置疑的是在Swift中，协议的功能被空前的强化。数组通过实现协议、以及协议的嵌套、拓展功能，具有了很多特性。数组的背后交织着错综复杂的协议、方法和拓展。

下面是数组的定义：

```swift
public struct Array<Element> : CollectionType, MutableCollectionType, _DestructorSafeContainer {
    public var startIndex: Int { get }
    public var endIndex: Int { get }
    public subscript (index: Int) -> Element
    public subscript (subRange: Range<Int>) -> ArraySlice<Element>
}
```

数组是一个结构体，它实现了三个协议，有两个变量和两个下标脚本。除此以外，数组还有大量的拓展。

### 数组拓展

数组的大量功能在拓展中完成，由于拓展很多，我就不列出完整代码，只是做一个整理。数组一共拓展了四类协议：

* `ArrayLiteralConvertible `： 这个协议是为了支持这样的语法的：`let a: Array<Int> = [1,2,3]`。实现协议很简单，只要提供一个自定义方法即可：

```swift
public init(arrayLiteral elements: Element...)
```
* `_Reflectable `：这个协议用于处理反射相关的内容，这里不做详解
* `CustomStringConvertible `和`CustomDebugStringConvertible `：这两个是方便我们调试的协议，与数组自身的功能无关。
* `_ArrayType `：这是数组**最关键的**部分。在实现这个协议之前，数组还只是一个普通的集合类型，它仅仅是拥有下标，而且可以重复遍历所有元素而已。而通过实现`_ArrayType `协议，它可以**在指定位置(下标)添加或移除一个或多个元素**，它还有了自己的`count`属性。

这一点也许有些颠覆我们此前的认识。一个集合类型，是不能添加或删除元素的。数组通过实现了`_ArrayType `协议提供了这样的功能。但这也很容易理解，因为集合的本质还是在于元素的收集，而非考虑如何改变这些元素。

`_ArrayType `协议还给了我们一个非常重要的启示。比如说我想实现自己的数据结构——栈，那么就应该实现对应的`_StackType `协议。这种协议要充分考虑数据结构自身的特点，从而提供对应的方法。比如我们不可能向栈的某个特定位置添加若干个元素(只能向栈顶添加一个)。所以`_StackType `协议中不会定义`append `、`appendContentsOf`这样的方法，而是应该定义`pop`和`push`方法。

# SequenceType

接下来的任务是搞清楚`ColectionType`的原理了。不过在此之前，我们先来看看`SequenceType`的一些细节，毕竟`CollectionType`协议是继承了`SequenceType`协议的。

在有了`Generator`之后，我们已经可以在`while`循环中用`Generator`的`next`方法遍历所有元素了。之前也说过，`SequenceType`使对元素的多次遍历成为可能。

注意，仅仅是成为可能而已。如果遍历的细节由`Generator`控制，那么多次遍历是没有问题的。在极个别情况下，但如果遍历的细节依赖于`SequenceType`自身的某个属性，而且这个属性会发生变化，那么就不能多次遍历所有元素了。

`SequenceType`协议的基本部分非常简单，只有一个`generator()`方法，封装了`Generator`的创建过程。

一旦有了遍历元素的概念，`SequenceType`立刻就有了非常多的拓展。这里简单列举几个比较关键的：

* `forEach`：这个拓展使得我们可以用for循环遍历集合了：`for item in sequence`
* `dropFirst(n: Int)`和`dropLast(n: Int)`：这两个方法返回的是除了前(后)n个元素之外的`Sequence`。需要提一下的是，由于此时的`SequenceType`还没有下标的概念，这两个方法的实现是**非常复杂**的。
* `prefix(maxLength: Int)`和`suffix(maxLength: Int)`：和刚刚两个方法类似，这两个方法返回的是前(后)maxLength个元素，实现也不简单。
* `elementsEqual`、`contains`、`minElement`、`maxElement`等，这些都是针对元素的判断和选择。
* `map`、`flatMap`、`filter`、`reduce`这些方法是针对所有元素的变换。

`SequenceType`的拓展实在是太多了，但总结来看不外乎两点：

1. 由于可以多次遍历元素了，我们可以对元素进行各种比较、处理、筛选等等操作。这些派生出来的方法和函数极大的强化了`SequenceType `的功能。
2. 由于`SequenceType `自身的局限性，不能保证一定可以多次遍历所有元素，还没有下标和元素位置的概念，因此某些方法的实现还不够高效，

带着这样的遗憾，我们来看看最关键的`CollectionType `是如何实现的。

# 细谈CollectionType

之前我们说过`CollectionType`协议是在`SequenceType`的基础上实现了`Indexable`协议。由于协议的继承关系，任何实现了`CollectionType`协议的类，必须实现`Indexable`协议规定的两个参数：`startIndex`和`endIndex`，以及一个下标脚本：`subscript (position: Self.Index) -> Self._Element { get }`。即使这三个要求在`CollectionType`中没有直接标出来。

回顾一下数组定义的前三行，正是满足了这三个要求。再看`CollectionType`,它不仅重载了`Indexable`的一个下标脚本，还额外提供了一个下标脚本用来访问某一段元素，这个下标脚本返回的类型是切片(`Slice`)。这也正是数组定义的第四行，实现的内容。

细心的读者可能已经注意到，`CollectionType`还定义了很多属性和方法，比如：`prefixUpTo`、`suffixFrom`、`isEmpty`、`first`等等。但数组没有实现其中的任何一个。

事实上，这不仅不是数组设计的失败之处，而正是Swift协议的强大之处。Swift可以通过协议拓展，为计算属性和方法提供默认实现。因此，数组可以不用写任何代码就具备这些方法。更赞的是，任何实现了`CollectionType`协议的类型也因此具有了这些方法。

观察一下`CollectionType `的其它拓展，大部分都是重写了`SequenceType `中的实现。之前已经提到过`SequenceType `没有下标的概念，而类似于`dropFirst `这样的方法，利用下标的概念是非常容易实现的。

除了对一些已有方法的重写之外，`CollectionType `还新增了一些基于下标的方法。比如`indexOf() `等。

套用官方文档中对`CollectionType `的总结就是：

>     A multi-pass sequence with addressable positions

也就是说`CollectionType `是可以多次遍历，元素可定位的`SequenceType `

# 总结

`Element`、`Generator`、`SequenceType`、`CollectionType`、`Array`由下至上构造了数组背后的层次结构。他们的关系如下图所示：

![Swift数组层次结构](http://images.bestswifter.com/Advanced%20Swift/Collection/array-hierarchy.png)


如果我们希望定义一个自己的数据结构，比如链表。首先可以明确它要实现`CollectionType`协议。链表应该是和`Array`同层次的类型。然后我们需要定义一个`_ListType`的协议，这个协议对应数组的`_ArrayList`协议，根据数据结构自身的特性定义一些方法。

如果觉得`CollectionType`甚至是`SequenceType`不满足我们的需求，我们还可以自己实现对应的类型。难度不会很大，因为它们虽然负责，但大多数方法已有默认实现，我们只要重写一些关键的逻辑即可。

最后需要说明的是，Swift中对集合的实现实在是太复杂了，如果每个都详细分析，怕是可以写一本书。希望读完这篇文章后，读者可以举一反三，自行阅读源码解决相关问题。