# 结论

首先把结论写在文章开头，因为接下来的分析会有些啰嗦、复杂，如果不愿意深究的话只要记住Swift中数组扩容的原理是：

>Swift2中创建一个空数组，默认容量为2，当长度和容量相等，且还需要再添加元素时，创建一个

>两倍长度于旧数组的新数组，把旧数组的元素拷贝过来，再把元素插入到新数组中。

# 引子

下面这段代码希望通过多线程技术，加速向数组中添加数字这个过程，我们来看看它有什么问题：

```swift
var array: [Int] = []
let concurrentQueue = dispatch_queue_create("com.gcd.kt", DISPATCH_QUEUE_CONCURRENT)

for i in 1...10000 {
  dispatch_async(concurrentQueue, { () -> Void in
    array.append(i)
  })
}
```

代码很简短，看上去问题不大。不过如果你运行完这段代码而且程序没有崩溃，我强烈建议买一份彩票，因为你的运气已经好到逆天了。

通常情况下，你会遇到这样的报错
> fatal error: UnsafeMutablePointer.destroy with negative count

# Append方法实现

程序会断在`array.append(i)`这一行。也就是`append `方法出了问题。我们知道Swift里的数组不像C语言，不需要提前定义好长度，更像是C++的`vector`和OC的`NSMutableArray`。

所以，会不会是数组的可变性，导致了`append`方法是线程不安全的呢，带着这样的猜想，我们来研究一下Swift是如何实现Append方法的。

Swift已经开源了，github上[相关源码](https://github.com/apple/apple)已经可以下载。

虽然明知道有些文件夹不会包含append方法的实现源码，但真想找到也不容易。如果你试着搜索"append"的话，相关文件非常多，因为"append"本身就是一个非常常用的单词。我采取的办法是搜索完整的函数定义，而函数定义是我们很容易看到的。

当我们搜索"mutating func append(newElement: Element)"后，就只有六个相关文件了。如图所示:

![搜索结果](http://images.bestswifter.com/Swift_append/search_result.png)

前三个文件无法直接打开，暂时先不管。其实第三个一看也知道是单元测试文件。第六个是字符串，也不是我们感兴趣的。所以我们依次打开"ArrayType.swift"和"RangeReplaceableCollectionType.swift"这两个文件。

>提示：这两个文件的目录都是"/swift/stdlib/public/core"

遗憾的是，ArrayType.swift文件中没有找到相关函数，RangeReplaceableCollectionType.swift文件倒是有一个`append`方法，不过参数类型对不上。于是我想到第一个文件——"Arrays.swift.gyb"，去掉gyb后缀后果然可以打开了。并且成功的找到了我们感兴趣的`append`方法：

```swift
public mutating func append(newElement: Element) {
  _makeUniqueAndReserveCapacityIfNotUnique()
  let oldCount = _getCount()
  _reserveCapacityAssumingUniqueBuffer(oldCount)
  _appendElementAssumeUniqueAndCapacity(oldCount, newElement: newElement)
}
```

# 源码分析

代码不长，我们逐行看一下

* 第一行的函数，看名字表示，如果这个数组不是惟一的，就把它变成惟一的，而且保留数组容量。在 Xcode 里面搜索一下这个函数：

```swift
internal mutating func _makeUniqueAndReserveCapacityIfNotUnique() {
  if _slowPath(!_buffer.isMutableAndUniquelyReferenced()) {
    copyToNewBuffer(_buffer.count)
  }
}
```

这个函数会进行一个判断判断——如果数组不是被唯一引用的，就会复制一份新的。这其实就是“**写时赋值(copy-on-write)**”技术。如果你想了解它的具体使用，可以参考我的这篇文章——[《第二章——集合（数组与可变性）》](http://www.jianshu.com/p/21d518da46ed)

不过对于文章开头那个例子的数组来说，它肯定是唯一引用的。所以`_copyToNewBuffer `函数不会调用，我们先记下这个方法。然后回到`append `方法继续研究。

* 第二行用一个变量`oldCount`保存数组当前长度。

* 第三行的函数表示，在假设当前数组是唯一引用的前提下，保留数组容量。之所以做出这样的假设，是因为此前已经调用过`_makeUniqueAndReserveCapacityIfNotUnique `方法，即使这个数组不是唯一引用，也被复制了一份新的。我们来看看`_reserveCapacityAssumingUniqueBuffer `方法的实现：

```swift
internal mutating func _reserveCapacityAssumingUniqueBuffer(oldCount : Int) {
  _sanityCheck(_buffer.isMutableAndUniquelyReferenced())

  if _slowPath(oldCount + 1 > _buffer.capacity) {
    _copyToNewBuffer(oldCount)
  }
}
```

第一行有一个`_sanityCheck `来判断数组是否可变且唯一引用。"sanity"说明这个判断是符合常理的，虽然它很有可能并没有效果，但也是为了确保万无一失。

下面还有一个判断，检查当前数组长度加一后是否大于数组容量。如果判断成立，说明`oldCount == _buffer.capacity`，在实际编程中，就意味着数组需要扩容了。可以看到又会执行刚刚提到过的`_copyToNewBuffer `函数。我们还是先把这个函数放一放，接着往后看。

* 最后一行的函数表示，假设数组是唯一引用的，且数组容量也设置正确，把新的元素添加到数组中。这其实是真正执行了“append”操作的地方。它的实现如下：

```swift
internal mutating func _appendElementAssumeUniqueAndCapacity(oldCount : Int, newElement : Element) {
  _sanityCheck(_buffer.isMutableAndUniquelyReferenced())
  _sanityCheck(_buffer.capacity >= _buffer.count + 1)

  _buffer.count = oldCount + 1
  (_buffer.firstElementAddress + oldCount).initialize(newElement)
}
```

首先是两个基本判断，然后把`count `属性加一，最后获取到将要添加的位置的地址，用一个新的值初始化它。


OK，`append`方法的结构基本上了解了，首先会保证数组是唯一引用的，然后处理数组的容量问题，最后把待插入的元素放到指定位置上。其中最关键，也是目前还没有彻底明白的一步，就是之前所说的`_copyToNewBuffer `函数了

# copyToNewBuffer

先来看看`copyToNewBuffer`函数的实现：

```swift
internal mutating func _copyToNewBuffer(oldCount: Int) {
  let newCount = oldCount + 1
  var newBuffer = Optional(
  _forceCreateUniqueMutableBuffer(&_buffer, countForNewBuffer: oldCount, minNewCapacity: newCount))
  _arrayOutOfPlaceUpdate(&_buffer, &newBuffer, oldCount, 0, _IgnorePointer())
}
```

这个方法又分为两步，`_forceCreateUniqueMutableBuffer `和`_arrayOutOfPlaceUpdate `。前者实现了新存储区域的创建，而后者完成了数据的复制工作。

为了简单起见，我们先看看`_arrayOutOfPlaceUpdate `函数,这个函数的实现太长了，不过好在有注释：

```swift
/// Initialize the elements of dest by copying the first headCount
/// items from source, calling initializeNewElements on the next
/// uninitialized element, and finally by copying the last N items
/// from source into the N remaining uninitialized elements of dest.
///
/// As an optimization, may move elements out of source rather than
/// copying when it isUniquelyReferenced.
```

大意是说，源数组中已存在的元素会被复制到目标数组中，如果新数组比较长，空缺部分会调用`initializeNewElements `方法来初始化。为了优化性能，被唯一引用的元素可能会直接从源数组移到新数组而不是复制。其实就是换了一个指针指向那个元素，从而避免了复制。

接下来我们再研究一下比较关键的`_forceCreateUniqueMutableBuffer `部分，也就是数组是怎样扩容的：

```swift
@inline(never)
func _forceCreateUniqueMutableBuffer<_Buffer : _ArrayBufferType>(inout source: _Buffer,  countForNewBuffer: Int,  minNewCapacity: Int) -> _ContiguousArrayBuffer<_Buffer.Element> {

  //其实什么也没干，多加了一个参数就转发给_forceCreateUniqueMutableBufferImpl 了
  return _forceCreateUniqueMutableBufferImpl(&source, countForBuffer: countForNewBuffer, minNewCapacity: minNewCapacity, requiredCapacity: minNewCapacity)
}

internal func _forceCreateUniqueMutableBufferImpl<_Buffer : _ArrayBufferType>(inout source: _Buffer,  countForBuffer: Int, minNewCapacity: Int, requiredCapacity: Int) -> _ContiguousArrayBuffer<_Buffer.Element> {
  _sanityCheck(countForBuffer >= 0)
  _sanityCheck(requiredCapacity >= countForBuffer)
  _sanityCheck(minNewCapacity >= countForBuffer)

  let minimumCapacity = max(requiredCapacity, minNewCapacity > source.capacity? _growArrayCapacity(source.capacity) : source.capacity)

  return _ContiguousArrayBuffer(count: countForBuffer, minimumCapacity: minimumCapacity)
}
```

`_forceCreateUniqueMutableBufferImpl `函数刚开始的三个检查读者可以自行理解，关键部分在于`minimumCapacity `的计算。因为它会作为容量参数被传到用于创建新的Buffer的函数中。

这个函数有四个参数，第一个参数buffer可以理解为数组中真正用于数据存放的那个部分。对于最后两个参数，意思有点像，我们不妨考虑一个实际的、需要进行数组扩容的情况，向一个容量为3，长度为3的数组新增一个元素1，此时函数的调用顺序如下：

1. `append(1)`
2. `_reserveCapacityAssumingUniqueBuffer(3)`
3. `_copyToNewBuffer(3)`
4. `_forceCreateUniqueMutableBuffer(&_buffer, countForNewBuffer: 3, minNewCapacity: 4)`
5. `_forceCreateUniqueMutableBufferImpl(&_buffer, countForBuffer: 3, minNewCapacity: 4, requiredCapacity: 4)`
        
还记得`_copyToNewBuffer()`方法里的`let newCount = oldCount + 1`么，所以`oldCount(=3)`作为`minNewCapacity `，而`newCount(=4)`作为`requiredCapacity `参数被传入`_forceCreateUniqueMutableBufferImpl `方法。

此时，`minimumCapacity`的计算，其实就是以下表达式的值：

```swift
max(4, 4 > 3 ? _growArrayCapacity(3) : 3)
```

我们知道，如果数组需要扩容，`source.capacity`总是等于`minNewCapacity `的。也就是说上式可以写为：

```swift
max(length+1, length+1 > length ? _growArrayCapacity(length) : length)

//等价于
max(length+1, _growArrayCapacity(length))
```

可以看到`_growArrayCapacity `返回值是传入参数的两倍：

```swift
@warn_unused_result
internal func _growArrayCapacity(capacity: Int) -> Int {
  return capacity * 2
}
```

所以`minimumCapacity` = `max(length+1, 2 * length)` = `2 * length`。也就是新扩容的数组长度其实翻倍了。

# 线程安全

现在我们可以理解为什么`append`方法不是线程安全的了。如果在某一个线程中插入新元素，导致了数组扩容，那么Swift会创建一个新的数组（意味着地址完全不同）。然后ARC会为我们自动释放旧的数组，但这时候可能另一个线程还在访问旧的数组对象。

# 验证

说了这么多，我们来证明一下Swift数组扩容的工作原理：

```swift
let semaphore = dispatch_semaphore_create(1)
var array: [Int] = []
for i in 1...100000 {
  array.append(i)
  let arrayPtr = UnsafeMutableBufferPointer<Int>(start: &array, count: array.count)
  print(arrayPtr)
}
```

运行结果如下，可以验证文章开头的结论：“初始容量为2，扩容时容量翻倍”

![容量翻倍](http://images.bestswifter.com/WX20180213-174100@2x.png)
