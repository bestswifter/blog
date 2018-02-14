本文翻译、整理自：[Exploring Swift Dictionary's Implementation](http://ankit.im/swift/2016/01/20/exploring-swift-dictionary-implementation/?utm_campaign=This%2BWeek%2Bin%2BSwift&utm_medium=email&utm_source=This_Week_in_Swift_71)

Swift中字典具有以下特点：

* 字典由两种范型类型组成，分别是Key（必须实现`Hashable `协议）和Value
* 提供一组Key和Value，可以向字典中插入一条新的数据
* 如果Key已经被插入字典，则可以通过Key获取到Value
* 可以通过Key删除一条字典中的数据
* 每个Key对应，且唯一对应字典中的一个Value

有很多种方式可以用于存储这些Key-Value对，Swift中字典采用了使用线性探测的开放寻址法。

我们知道，哈希表不可避免会出现的问题是哈希值冲突，也就是两个不同的Key可能具有相同的哈希值。线性探测是指，如果出现第二个Key的哈希值和第一个Key的哈希值冲突，则会检查第一个Key对应位置的后一个位置是否可用，如果可用则把第二个Key对应的Value放在这里，否则就继续向后寻找。

一个容量为8的字典，它实际上只能存储7个Key-Value对，这是因为字典需要至少一个空位置作为插入和查找过程中的停止标记。我们把这个位置称为“**洞**”。

举个例子，假设Key1和Key2具有相同的哈希值，它们都存储在字典中。现在我们查找Key3对应的值。Key3的哈希值和前两者相同，但它不存在于字典中。查找时，首先从Key1所在的位置开始比较，因为不匹配所以比较Key2所在的位置，而且从理论上来说只用比较这两个位置即可。如果Key2的后面是一个洞，就表示查找到此为止，否则还得继续向后查找。

在实际内存中，它的布局看上去是这样的：

![布局-1](http://images.bestswifter.com/Dictionary/memory.png)

创建字典时会分配一段连续的内存，其大小很容易计算：

`size = capacity * (sizeof(Bitmap) + sizeof(Keys) + sizeof(Values))`

从逻辑上来看，字典的组成结构如下：

![逻辑布局](http://images.bestswifter.com/Dictionary/logical.png)

其中每一列称为一个**bucket**，其中存储了三样东西：位图的值，Key和Value。bucket的概念其实已经有些类似于我们实际使用字典时，Key-Value对的概念了。

bucket中位图的值用于表示这个bucket中的Key和Value是否是已初始化且有效的。如果不是，那么这个bucket就是一个洞。

介绍完以上基本概念后，我们由底层向高层介绍字典的实现原理：

## _HashedContainerStorageHeader（结构体）

![_HashedContainerStorageHeader](http://images.bestswifter.com/Dictionary/_HashedContainerStorageHeader.png)

这个结构体是字典所使用内存的头部，它有三个成员变量：

* capacity：字典的容量，表示字典当前最多可以存储多少Key-Value对
* count：字典中元素数量，表示字典当前实际存储的Key-Value对的数量
* maxLoadFactorInverse：当字典需要扩容时使用到的因子，新的capacity是旧的capacity乘以这个因子。

## _NativeDictionaryStorageImpl<Key, Value>（类）

这个类是`ManagedBuffer<_HashedContainerStorageHeader, UInt8>`的子类。

这个类的作用是为字典分配需要使用的内存，并且返回指向位图、Key和Value数组的指针。比如：

```swift
internal var _values: UnsafeMutablePointer<Value> {
  @warn_unused_result
  get {
    let start = UInt(Builtin.ptrtoint_Word(_keys._rawValue)) &+
      UInt(_capacity) &* UInt(strideof(Key.self))
    let alignment = UInt(alignof(Value))
    let alignMask = alignment &- UInt(1)
    return UnsafeMutablePointer<Value>(
        bitPattern:(start &+ alignMask) & ~alignMask)
  }
}
```

由于位图、Key和Value数组所在的内存是连续分配的，所以Value数组的指针`values_pointer `等于`keys_pointer + capacity * keys_pointer`。

分配字典所用内存的函数和下面的知识关系不大，所以这里略去不写，有兴趣的读者可以在原文中查看。

在分配内存的过程中，位图数组中所有的元素值都是0，这就表示所有的bucket都是洞。另外需要强调的一点是，到目前为止(分配字典所用内存)范型Key不必实现`Hashable`协议。

目前，字典的结构组成示意图如下：

![_NativeDictionaryStorageImpl](http://images.bestswifter.com/Dictionary/_NativeDictionaryStorageImpl.png)

## _NativeDictionaryStorage<Key : Hashable, Value>（结构体）

这个结构体将`_NativeDictionaryStorageImpl `结构体封装为自己的`buffer`属性，它还提供了一些方法将实际上有三个连续数组组成的字典内存转换成逻辑上的bucket数组。而且，这个结构体将bucket数组中的第一个bucket和最后一个bucket在逻辑上链接起来，从而形成了一个bucket环，也就是说当你到达bucket数组的末尾并且调用`next`方法时，你又会回到bucket数组的开头。

在进行插入或查找操作时，我们需要算出这个Key对应哪个bucket。由于Key实现了`Hashable`，所以它一定实现了`hashValue`方法并返回一个整数值。但这个哈希值可能比字典容量还大，所以我们需要压缩这个哈希值，以确保它属于区间`[0, capacity)`：

```swift
@warn_unused_result
internal func _bucket(k: Key) -> Int {
  return _squeezeHashValue(k.hashValue, 0..<capacity)
}
```

通过`_next `和`_prev `函数，我们可以遍历整个bucket数组，这里虽然使用了溢出运算符，但实际上并不会发生溢出，个人猜测是为了性能优化：

```swift
internal var _bucketMask: Int {
  return capacity &- 1
}

@warn_unused_result
internal func _next(bucket: Int) -> Int {
  return (bucket &+ 1) & _bucketMask
}

@warn_unused_result
internal func _prev(bucket: Int) -> Int {
  return (bucket &- 1) & _bucketMask
}
```

字典容量`capacity`一定可以表示为2的多少次方，因此`_bucketMask `这个属性如果用二进制表示，则一定全部由1组成。举个例子体验一下，假设`capacity = 8`：

* bucket = 6，调用_next方法，返回值为 7 & 7，也就是7.
* bucket = 7，调用_next方法，返回值为 8 & 7，二进制表示为1000 & 0111，因此返回值为0。也就是返回了数组的起始位置。
* bucket = 0，调用_prev方法，返回值为 -1 & 7，二进制表示为1…1111 & 0…0111，因此返回值为111，也就是7，回到了数组的结束位置。

在插入一个键值对时，我们首先计算出Key对应哪个bucket，然后调用下面的方法把Key和Value写入到bucket中，同时把位图的值设置为true：

```swift
@_transparent
internal func initializeKey(k: Key, value v: Value, at i: Int) {
  _sanityCheck(!isInitializedEntry(i))

  (keys + i).initialize(k)
  (values + i).initialize(v)
  initializedEntries[i] = true
  _fixLifetime(self)
}
```

另一个需要重点介绍的函数是`_find`：

* `_find`函数用于找到Key对应的bucket
* 需要指定需要指定从哪个bucket开始寻找，因此需要`_buckey(key)`函数的配合
* 如果参数key和某个bucket中的Key匹配，则返回这个bucket的位置
* 如果没有找到，则返回接下来的第一个洞，表示key可以插入到这里
* 通过位图判断当前bucket是不是一个洞
* 这种算法被称为线性探测

```swift
@warn_unused_result
internal
func _find(key: Key, _ startBucket: Int) -> (pos: Index, found: Bool) {
  var bucket = startBucket
  while true {
    let isHole = !isInitializedEntry(bucket)
    if isHole {
      return (Index(nativeStorage: self, offset: bucket), false)
    }
    if keyAt(bucket) == key {
      return (Index(nativeStorage: self, offset: bucket), true)
    }
    bucket = _next(bucket)
  }
}
```

* 一般来说，`_squeezeHashValue `函数的返回值就是Key对应的bucket的下标，不过需要考虑不同的Key哈希值冲突的情况。
* 在这种情况下，`_find`函数会找到下一个可用的洞，以便插入数据。

## hashValue优化

`_squeezeHashValue `函数的本质是对Key的哈希值再次求得哈希值，而一个优秀的哈希函数是提高性能的关键。`_squeezeHashValue `函数基本上符合要求，不过目前惟一的缺点是哈希变换的种子还是一个占位常量，有兴趣的读者可以阅读完整的函数实现，其中的`seed `就是一个值为`0xff51afd7ed558ccd `的常量：

```swift
func _squeezeHashValue(hashValue: Int, _ resultRange: Range<UInt>) -> UInt {
  let mixedHashValue = UInt(bitPattern: _mixInt(hashValue))
  let resultCardinality: UInt = resultRange.endIndex - resultRange.startIndex
  if _isPowerOf2(resultCardinality) {
    return mixedHashValue & (resultCardinality - 1)
  }
  return resultRange.startIndex + (mixedHashValue % resultCardinality)
}

func _mixUInt64(value: UInt64) -> UInt64 {
  // Similar to hash_4to8_bytes but using a seed instead of length.
  let seed: UInt64 = _HashingDetail.getExecutionSeed()
  let low: UInt64 = value & 0xffff_ffff
  let high: UInt64 = value >> 32
  return _HashingDetail.hash16Bytes(seed &+ (low << 3), high)
}

static func getExecutionSeed() -> UInt64 {
  // FIXME: This needs to be a per-execution seed. This is just a placeholder
  // implementation.
  let seed: UInt64 = 0xff51afd7ed558ccd
  return _HashingDetail.fixedSeedOverride == 0 ? seed : fixedSeedOverride
}

static func hash16Bytes(low: UInt64, _ high: UInt64) -> UInt64 {
  // Murmur-inspired hashing.
  let mul: UInt64 = 0x9ddfea08eb382d69
  var a: UInt64 = (low ^ high) &* mul
  a ^= (a >> 47)
  var b: UInt64 = (high ^ a) &* mul
  b ^= (b >> 47)
  b = b &* mul
  return b
}
```

目前，字典的结构总结如下：

![](http://images.bestswifter.com/Dictionary/summary1.png)

## _NativeDictionaryStorageOwner（类）

这个类被用于管理字典的引用计数，以支持写时复制(COW)特性。由于`Dictionary `和`DictionaryIndex `都会引用实际存储区域，所以引用计数为2。不过写时复制的唯一性检查不考虑由`DictionaryIndex `导致的引用，所以如果字典通过引用这个类的实例对象来管理引用计数值，问题就很容易处理。

```swift
/// This class is an artifact of the COW implementation.  This class only
/// exists to keep separate retain counts separate for:
/// - `Dictionary` and `NSDictionary`,
/// - `DictionaryIndex`.
///
/// This is important because the uniqueness check for COW only cares about
/// retain counts of the first kind.

/// 这个类用于区分以下两种引用：
/// - `Dictionary` and `NSDictionary`,
/// - `DictionaryIndex`.
/// 这是因为写时复制的唯一性检查只考虑第一种引用
```

现在，字典的结构变得有些复杂，难以理解了：

![](http://images.bestswifter.com/Dictionary/awkward.png)

## _VariantDictionaryStorage<Key : Hashable, Value> (枚举)

这个枚举类型中有两个成员，它们各自具有自己的关联值，分别表示Swift原生的字典和Cocoa的字典：

```swift
case Native(_NativeDictionaryStorageOwner<Key, Value>)
case Cocoa(_CocoaDictionaryStorage)
```

这个枚举类型的主要功能是：

1. 根据字典的不同类型（原生 or Cocoa）执行对应的增删改查函数
2. 如果字典已经满了，则扩容
3. 更新或初始化Key-Value对：
	
	```swift
	internal mutating func nativeUpdateValue(
		value: Value, forKey key: Key
	) -> Value? {
		var (i, found) = native._find(key, native._bucket(key))
	    
		let minCapacity = found
		  ? native.capacity
		  : NativeStorage.getMinCapacity(
		      native.count + 1,
		      native.maxLoadFactorInverse)
		
		let (_, capacityChanged) = ensureUniqueNativeStorage(minCapacity)
		if capacityChanged {
		  i = native._find(key, native._bucket(key)).pos
		}
		
		let oldValue: Value? = found ? native.valueAt(i.offset) : nil
		if found {
		  native.setKey(key, value: value, at: i.offset)
		} else {
		  native.initializeKey(key, value: value, at: i.offset)
		  native.count += 1
		}
		
		return oldValue
	}
	```

4. 如果移除某个Key-Value对，就会在原地留下一个洞。下一次线性查找时有可能会提前停止，为了解决这个问题，我们需要在移除Key-Value对后，移动另一个Key-Value对补上这个洞，源码如下：

	```swift
	/// - parameter idealBucket: The ideal bucket for the element being deleted.
	/// - parameter offset: The offset of the element that will be deleted.
	/// Requires an initialized entry at offset.
	internal mutating func nativeDeleteImpl(
  		nativeStorage: NativeStorage, idealBucket: Int, offset: Int
	) {
		_sanityCheck(
	      nativeStorage.isInitializedEntry(offset), "expected initialized entry")
	
		// remove the element
		nativeStorage.destroyEntryAt(offset)
		nativeStorage.count -= 1
	
		// If we've put a hole in a chain of contiguous elements, some
		// element after the hole may belong where the new hole is.
		var hole = offset
	
		// Find the first bucket in the contiguous chain
		var start = idealBucket
		while nativeStorage.isInitializedEntry(nativeStorage._prev(start)) {
			start = nativeStorage._prev(start)
		}
	
		// Find the last bucket in the contiguous chain
		var lastInChain = hole
		var b = nativeStorage._next(lastInChain)
		while nativeStorage.isInitializedEntry(b) {
			lastInChain = b
			b = nativeStorage._next(b)
		}
	
		// Relocate out-of-place elements in the chain, repeating until
		// none are found.
		while hole != lastInChain {
			// Walk backwards from the end of the chain looking for
			// something out-of-place.
			var b = lastInChain
			while b != hole {
				let idealBucket = nativeStorage._bucket(nativeStorage.keyAt(b))
	
				// Does this element belong between start and hole?  We need
				// two separate tests depending on whether [start,hole] wraps
				// around the end of the buffer
				let c0 = idealBucket >= start
				let c1 = idealBucket <= hole
				if start <= hole ? (c0 && c1) : (c0 || c1) {
					break // Found it
				}
				b = nativeStorage._prev(b)
			}
	
			if b == hole { // No out-of-place elements found; we're done adjusting
				break
			}
	
			// Move the found element into the hole
			nativeStorage.moveInitializeFrom(nativeStorage, at: b, toEntryAt: hole)
			hole = b
		}
	}
	```

这段代码理解起来可能比较费力，我想举一个例子来说明就比较简单了，假设一开始有8个bucket，bucket中的value就是bucket的下标，最后一个bucket是洞：

```swift
Bucket数组中元素下标:  {0, 1, 2, 3, 4, 5, 6, 7(Hole)}
bucket中存储的Value:  {0, 1, 2, 3, 4, 5, 6, null}
```

接下来我们删除第五个bucket，这会在原地留下一个洞：

```swift
Bucket数组中元素下标:  {0, 1, 2, 3, 4(Hole), 5, 6, 7(Hole)}
bucket中存储的Value:  {0, 1, 2, 3,        , 5, 6         }
```

为了补上这个洞，我们把最后一个bucket中的内容移到这个洞里，现在第五个bucket就不是洞了：

```swift
Bucket数组中元素下标:  {0, 1, 2, 3, 4, 5, 6(Hole), 7(Hole)}
bucket中存储的Value:  {0, 1, 2, 3, 6, 5,        ,        }
```

![枚举](http://images.bestswifter.com/Dictionary/enum.png)

## 字典的完整结构

`Dictionary`结构体持有一个`_VariantDictionaryStorage`类型的枚举，作为自己的成员属性，所以整个字典完整的组成结构如下图所示：


![Swift字典结构](http://images.bestswifter.com/Dictionary/dictionary.png)