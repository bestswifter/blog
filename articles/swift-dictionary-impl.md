# 浅谈 Swift Dictionary

## 背景

在最近排查问题的过程中，发现经常需要和字典的汇编码打交道。在阅读汇编代码的过程中，由于对字典的内存布局不够了解，而对应的汇编码又经过了高度优化，因此排查成本极高。经过一段时间的学习和挣扎才勉强能看懂代码，再加上笔者一直对 Swift 源码比较感兴趣，因此谨以此文记录并且交流下学习过程。

俗话说授人以鱼不如授人以渔，本文只会分析字典的下标访问方法的源码调试和汇编实现，希望起到抛砖引玉额效果。

在[四年前的博客](../articles/swift-dictionary.md) 中，我从代码层面介绍了 Swift 字典的实现。四年过去了，Swift 字典的实现没有发生很大的改动，因此这篇文章也还具备一定的参考价值。

## 字典的内存布局

由于汇编中存在较多对字节内存布局的直接操作，所以在开始学习前有必要简单介绍下字典的内部布局，在源码中其实已经有了[明确的描述](https://github.com/apple/swift/blob/main/stdlib/public/core/Dictionary.swift#L33)。

如果不考虑兼容 ObjC 中的 `NSDictionary` 的场景，Swift 原生的字典最终内部会存储一个 `__RawDictionaryStorage` 类的实例。我们知道类实例在 `0x0` 和 `0x8` 的偏移分别是它的 isa 指针和引用计数，因此真正的成员变量是从 `0x10` 开始布局的。它们分别是：

* 0x10: count，表示字典内部键值对的数量
* 0x18: capacity，表示在不触发扩容之前，当前最多能存储多少个键值对，如果存不下会触发扩容的流程。
* 0x20: _scale，中文翻译叫规模，这是字典**非常关键的一个属性**，2 的 scale 次方(后面记为 `2^scale`) 是 bucket 的个数，也就是物理上”真正的”最多能存储多少个键值对，只不过一般到达 `capacity` 的时候就会触发扩容。它的类型是 `Int8`，因此只占一个字节
* 0x21：_reservedScale，暂时用不到，类型也是 `Int8`，占一个字节
* 0x22: _extra，源码注释明确说了没用，类型是 `Int16`，占两个字节
* 0x24: _age，暂时用不到，类型是 `Int32`，占四个字节
* 0x28: _seed，做哈希函数时用到种子
* 0x30: _rawKeys，存储 Key 的指针，下文会有介绍
* 0x38: _rawValues，存储 Value 的指针，下文会有介绍

这里最重要几个属性，分别是 count(+0x10)、scale(+0x20)、rawKeys(+0x30) 和 rawValues(+0x38).

上述是 `__RawDictionaryStorage` 实例的成员变量的布局结构，实际上从 `0x40` 的偏移位置开始，才是真正存放的数据：

### 位图
从 `0x40` 的偏移开始存放的是实例的 `bitset`，中文可以翻译为位图。它定义在源码的 `stdlib/public/core/Bitset.swift` 文件中:

```swift
internal struct _UnsafeBitset {
    internal let words: UnsafeMutablePointer<Word>

    internal let wordCount: Int

    internal init(words: UnsafeMutablePointer<Word>, wordCount: Int) {
        self.words = words
        self.wordCount = wordCount
    }
}
```

有 C 语言基础的同学应该可以看出，它存储了一个 `Word` 的指针和数量，其实就是一个 `Word` 结构的数组。

至于 `Word` 的结构就更加简单了：

```swift
internal struct Word {
    @usableFromInline
    internal var value: UInt

    @inlinable
    internal init(_ value: UInt) {
        self.value = value
    }
 }
```

它本质上就是一个 `UInt`，特殊之处在于这个 `UInt` 一共 64 位，每一位都可以用来表示一个对应位置是否被占用。这样的一位被称为一个 Bucket（桶）。

上面已经介绍过，Bucket 的数量一定是 2 的整数次方，这个整数就用 `scale` 表示。

举个例子，当 `scale = 7` 时，共有 `2 ^ 7 = 128` 个 Bucket，需要两个 Word 存储。此时实例的 `+0x40` 和 `+0x48` 的位置就是由两个 `Word` 组成的一个 `bitset`

### Keys & Values

紧随位图之后的内存区域是真正存储 Keys 的内存地址。它有点类似于 Key 的数组，不过各个 Key 并不是连续存储，而是根据它所在的 Buck 决定存储在哪个位置。这块内存地址的总大小是 `2^scale * sizeof(Key)`，没有存放 Key 的位置可以理解为空。

这块内存的起始位置，可以通过 `scale` 计算出 Bucket 的数量，从而计算出 `Word` 的数量和 `Bitset` 的大小得出。也可以直接看 `+0x30` 偏移上的 `_rawKeys` 指针。

在 Keys 的内存地址之后就是 Values 的内存，和 Keys 的存储方式是完全一样的。它的大小是 `2^scale * sizeof(Value)`，起始位置可以动态计算，也可以直接读取 `+0x38` 偏移上的 `_rawValues` 指针。

## 下标函数的汇编实现

至此我们已经清楚的掌握了字典的内存布局，接下来会通过阅读一段源码来加深理解。

首先构造一段测试代码，并且在 Release 模式下进行编译：

```swift
@inline(never)
func find(dic: [String: Int], key: String) -> Int? {
    return dic[key]
}
```

对应的汇编实现如图所示：

![](https://images.bestswifter.com/uPic/call-find.png)

这里的参数 `dic` 通过 `$rdi` 寄存器传入函数，首先检查 `+0x10` 的偏移是不是 0。如果是 0 就会走右侧绿色分支，返回 `nil`。这段很好理解，前面已经分析过 `+0x10` 的位置存储了 `count`，如果 `count = 0 ` 表示空字典是不需要再做任何处理的。

如果字典不为空，会调用 `generic specialization <Swift.String> of Swift.__RawDictionaryStorage.find` 函数。这是 `find` 函数的范型特化版本，内部的实现稍后分析，它会返回一个 bucket 和 Bool 标记位，用来表示是否找到。如果找不到也是返回 `nil`。

如果能找到，那么 `bucket` 的值会存储在 `$rax` 寄存器中，通过下面的方式去读取：

```asm
; r13 + 0x38 是 _rawValues 的指针
; 那么 [r13 + 0x38] 就是得到了 _rawValues 实际的内存地址
mov        rcx, qword [r13+0x38]

; 类似于数组的下标访问，rcx 是起点，rax 是 bucket，0x8 是 Int 的大小
; 把读取到的的值存储到 r15 寄存器
mov        r15, qword [rcx+rax*8]
```

这样的取值逻辑是符合此前的内存布局分析的。

### 外层 find 函数的实现

这里我们详细分析一下 `__RawDictionaryStorage` 的 `find` 函数。

![](https://images.bestswifter.com/uPic/find-out-impl.png)

它对应的源码：

```swift
@inline(never)
internal final func find<Key: Hashable>(_ key: Key) -> (bucket: _HashTable.Bucket, found: Bool) {
    return find(key, hashValue: key._rawHashValue(seed: _seed))
}
```

这段函数比较简单，它会计算 `key` 的哈希值，然后传入内部的 `find` 函数

可以看到 `Hasher` 的初始化参数 `seed` 来自于字典本身 `+0x28` 偏移的位置，这也符合之前的内存布局分析

### 底层 find 函数的实现

这个函数的源码如下：

```swift
internal final func find<Key: Hashable>(_ key: Key, hashValue: Int) -> (bucket: _HashTable.Bucket, found: Bool) {
    let hashTable = _hashTable
    var bucket = hashTable.idealBucket(forHashValue: hashValue)
    while hashTable._isOccupied(bucket) {
        if uncheckedKey(at: bucket) == key {
            return (bucket, true)
        }
        bucket = hashTable.bucket(wrappedAfter: bucket)
    }
    return (bucket, false)
}
```

对应的汇编实现比较复杂，我们分段来看。分析逻辑通过注释的形式给出：

```asm
; 首先明确参数：
; 第一个入参是 String 类型，使用 rdi、rsi 两个寄存器传参
; 第二个参数是 hashValue，类型是 Int，使用 rdx 寄存器传参
mov        rbx, rdx

; cl 是 _scale，Int8 类型，所以用低 8 位就可以表示
mov        cl, byte [r13+0x20]

; r14 = 2^scale - 1
; 这个 r14 是在计算 bucketMast
mov        r14, 0xffffffffffffffff
shl        r14, cl
not        r14

; rbx: hashValue & bucketMast
; 这里是在计算 idealBucket
and        rbx, r14
```

这段汇编对应了 `hashTable.idealBucket` 函数，作用是计算出这个哈希值预期要存放的 Bucket：

```swift

internal struct _HashTable {
    internal var words: UnsafeMutablePointer<Word>

    internal let bucketMask: Int

    internal init(words: UnsafeMutablePointer<Word>, bucketCount: Int) {
        self.words = words
        self.bucketMask = bucketCount &- 1
    }

    internal func idealBucket(forHashValue hashValue: Int) -> Bucket {
        return Bucket(offset: hashValue & bucketMask)
    }
}
```

为什么 `hashValue & (2^scale - 1)` 就是预期的 Bucket 呢？其实既是一个小算法，我们知道 `2^scale - 1` 其实就是 `scale` 个 1 组成的二进制，与这个数字做按位与运算，就是只保留这个 hashValue 的低 scale 位。

再看接下来的一段汇编：

```asm
; rax 是 idealBucket / 64，也就是 word
shr        rax, 0x6

; r13 + 0x40 是 bitset 的起始地址，再加上 rax*8 是第 rax 个 word 的地址
; 对应源码中的 words[bucket.word]
mov        rax, qword [r13+rax*8+0x40]

; 获取对应 word 的第 bt 位，对应 _isOccupied 函数
; 注意这里的 rbx 可能大于 64，但是 bt 指令的参数会自动对 rbx 做模 64 的运算
; 因此这里 rbx 实际上对应代码里的 bucket.bit
; bt 指令和后面的 jae 跳转判断，对应 uncheckedContains 函数
bt         rax, rbx
       
; bit = 0 -> CF = 0 -> 走绿色分支，直接返回，说明此时 Bucket 没有被占用
jae        loc_100003d78
```

这一段对应着 _isOccupied 函数：

```swift
internal func _isOccupied(_ bucket: Bucket) -> Bool {
    return words[bucket.word].uncheckedContains(bucket.bit)
}

internal struct Bucket {
    internal var offset: Int

    internal var word: Int {
        return _UnsafeBitset.word(for: offset)
    }

    internal var bit: Int {
        return _UnsafeBitset.bit(for: offset)
    }
}

extension _UnsafeBitset {
    internal static func word(for element: Int) -> Int {
        // Note: We perform on UInts to get faster unsigned math (shifts).
        let element = UInt(bitPattern: element)
        let capacity = UInt(bitPattern: Word.capacity)
        return Int(bitPattern: element / capacity)
    }

    internal static func bit(for element: Int) -> Int {
        // Note: We perform on UInts to get faster unsigned math (masking).
        let element = UInt(bitPattern: element)
        let capacity = UInt(bitPattern: Word.capacity)
        return Int(bitPattern: element % capacity)
    }
}
```

看起来比较长，其实核心思想就是：对于一个 `Bucket` 来说它的 `word` 和 `bit` 分别是自身的值除以 `capacity` 的值和余数（也就是模）。检查一个 `Bucket` 是否占用，也就是检查第 `word` 个 `bit` 上的值是不是 1。

如果被占用了，就要把这一位的 Key 取出来，看看是不是和参数相同，因为这一位也可能是在做哈希时，因为产生了碰撞而被迫放到了这里。

```asm
; r15 是 _rawKeys
mov        r15, qword [r13+0x30]

; rbx 是 idealBucket
mov        rax, rbx

; rax = idealBucket * 16
; 这是因为一个 String 占 16 字节
shl        rax, 0x4

; 此时 r15 + rax 是 Keys 数组在下标为 bucket 的位置的值了
; 对应 uncheckedKey
mov        rdi, qword [r15+rax]
mov        rsi, qword [r15+rax+8]
```

对应的 `uncheckedKey` 函数非常简单，就是取下标：

```swift
internal final func uncheckedKey<Key: Hashable>(at bucket: _HashTable.Bucket) -> Key {
    return keys[bucket.offset]
}
```

接下来跳过一段字符串判等的逻辑，当不相等时，来到最后一个函数，获取下一个 Bucket：

```asm
; rbx 是 bucket，先 +1
; 然后再按位与上 bucketMask 得到新的 bucket
add        rbx, 0x1
and        rbx, r14
```

对应的 Swift 源码：

```swift
internal func bucket(wrappedAfter bucket: Bucket) -> Bucket {
    // The bucket is less than bucketCount, which is power of two less than
    // Int.max. Therefore adding 1 does not overflow.
    return Bucket(offset: (bucket.offset &+ 1) & bucketMask)
}
```

以上就是 `_find` 函数内部所有的关键逻辑分析

## 调试 stdlib 源码

最后，除了直接对照源码和汇编实现以外，我们还可以直接调试 Swift 源码，从另一个角度来理解 Swift 源码的工作原理。

这方面的资料比较多，这里就简单描述下具体步骤和我遇到的一些细节问题：

1. 首先创建一个 `swift-project` 目录，这个目录用于编译 Swift 源码
2. 在 `swift-project` 目录下下载 Swift 源码，执行命令：`git clone https://github.com/apple/swift.git`
3. 进入 `swift-project/swift` 目录，执行命令：`./utils/update-checkout --clone` 下载 Swift 的依赖。
4. 下载万完成后可以在 `swift-project` 目录下看到出现了很多和 `swift` 同级的目录
5. 进入 `swift-project/swift` 目录，执行开始编译源码：`/swift/utils/build-script -r --debug-swift-stdlib --lldb`
6. 编译完成后用 `VSCode` 打开 `swift-project` 目录，并且安装 CodeLLDB 插件
7. 在调试页面，创建自定义的 `launch.json` 文件：

```json
{
    "version": "0.2.0", 
    "configurations": [
        {
        "type": "lldb",
        "request": "launch",
        "name": "Debug",
        "program": "${workspaceFolder}/build/Ninja-RelWithDebInfoAssert+stdlib-DebugAssert/swift-macosx-x86_64/bin/swift", 
        "args": ["path/to/your/swift-file"],
        "cwd": "${workspaceFolder}"
        } 
    ]
}
```

这里需要注意的是，很多网上教程贴出的配置 `args` 都是空的，这样会导致 Swift 进入命令行解释模式(`REPL`)，但是这个能力已经被禁止了。所以需要把自己需要被执行的 Swift 文件作为参数填进来。

然后在自己感兴趣的 Swift 源码部分打上断点，启动调试就可以了。效果如图所示：

![](https://images.bestswifter.com/uPic/debug.png)

其实笔者还是更想用 Xcode 进行调试。无奈网上的资料表示 VSCode 的支持程度更好，更推荐。使用 Xcode 调试的文章也比较老，尝试了一下没有成功。只能再研究下，后续成功了再做分享了。

## 总结

本文可以主要分为三个部分：

1. 介绍字典的内存布局，主要关注 `count`、`scale`、`rawKeys/Values` 这些固定布局结构和 `bitset`、`Keys/Values` 这些不固定布局的结构。
2. 介绍下标方法的实现，包括源码层实现与汇编层实现，深入理解字典相关的概念和函数的原理
3. 简单介绍动态调试 Swift 源码的方案

希望上述方案能够帮助读者更好的理解字典和其它 Swift 的源码。