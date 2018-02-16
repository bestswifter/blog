在使用 `Block` 时，除了使用 `__weak` 修饰符避免循环引用外，还有一点经常容易忘记。苹果把它称为：“Strong-Weak Dance”。

# 问题来源

这是一种 强引用 --> 弱引用 --> 强引用 的变换过程。在弄明白为什么要如此大费周章之前，我们首先来看看一般的写法会有什么问题。

```objc
__weak MyViewController *wself = self;
self.completionHandler = ^(NSInteger result) {
  [wself.property removeObserver: wself forKeyPath:@"pathName"];
};
```

这种写法可以避免循环引用，但是我们要考虑这样的问题：

**假设 `block` 被放在子线程中执行，而且执行过程中 `self` 在主线程被释放了。由于 `wself ` 是一个弱引用，因此会自动变为 `nil`。而在 KVO 中，这会导致崩溃。**

# Strong-Weak Dance

解决以上问题的方法很简单，新增一行代码即可：

```objc
__weak MyViewController *wself = self;
self.completionHandler = ^(NSInteger result) {
  __strong __typeof(wself) sself = wself; // 强引用一次
  [sself.property removeObserver: sself forKeyPath:@"pathName"];
};
```

这样一来，`self` 所指向对象的引用计数变成 2，即使主线程中的 `self` 因为超出作用于而释放，对象的引用计数依然为 1，避免了对象的销毁。

# 思考

在和小伙伴的讨论过程中，他提出了几个问题。虽然都不难，但是有利于把各种知识融会贯通起来。

1. Q：下面这行代码，将一个弱引用的指针赋值给强引用的指针，可以起到强引用效果么？

	```objc
	__strong __typeof(wself) sself = wself;
	```
	
	A：会的。引用计数描述的是对象而不是指针。这句话的意思是：

	> sself 强引用 wself 指向的那个对象

	因此对象的引用计数会增加一个。

2. Q：`block` 内部定义了`sself`，会不会因此强引用了 `sself`？

	A：不会。`block` 只有截获外部变量时，才会引用它。如果是内部新建一个，则没有任何问题。

3. Q：如果在 `block` 内部没有强引用，而是通过 `if` 判断，是不是也可以，比如这样写：

	```objc
	__weak MyViewController *wself = self;
	wself.completionHandler = ^(NSInteger result) {
		if (wself) { // 只有当 wself 不为 nil 时，才执行以下代码
			[wself.property removeObserver: wself forKeyPath:@"pathName"];
		}
	};
	```
	
	A：不可以！考虑到多线程执行，也许在判断的时候，`self` 还没释放，但是执行 `self` 里面的代码时，就刚好释放了。

4. Q：那按照这个说法，`block` 内部强引用也没用啊。也许 `block` 执行以前，`self` 就释放了。

	A：有用！如果在 `block` 执行以前，`self` 就释放了，那么 `block` 的引用计数降为 0，所以自己就会被释放。这样它根本就不会被执行。另外，如果执行一个为 `nil` 的闭包会导致崩溃。

5. Q：如果在执行 `block` 的过程中，`block` 被释放了怎么办？

	A：简单来说，`block` 还会继续执行，但是它捕获的指针会具有不确定的值，详细内容请参考[这篇文章](http://stackoverflow.com/questions/12272783/what-happens-when-a-block-is-set-to-nil-during-its-execution)


# @strongify 和 @weakify

这是 [ReactiveCocoa](https://github.com/ReactiveCocoa/ReactiveCocoa) 中定义的一个宏。一般可以这样使用：

```objc
@weakify(self);
self.completionHandler = ^(NSInteger result) {
	@strongify(self);
	[self.property removeObserver: sself forKeyPath:@"pathName"];
};
```

本文并非分析它们的实现原理，所以就简单解释两点：

1. 这里的“@”没有任何用处，仅表示强调，这个宏实际上包含了一个空的 `AutoreleasePool`，这也就是为什么一定要加上“@”。

2. 它的原理还是和之前一样，生成了一段形如 `__weak MyViewController *wself = self;` 这种格式的代码：

	```objc
	#define rac_strongify_(INDEX, VAR) \\
    __strong __typeof__(VAR) VAR = metamacro_concat(VAR, _weak_);
	```

# Swift 中的情况

感谢 [@Cyrus_dev](http://www.jianshu.com/users/18729f511551/latest_articles) 的提醒，在 Swift 中也有 Strong-Weak Dance 的概念。最简单的方法就是直接沿用 OC 的思路：


```swift
self.completionHandler = { [weak self] in
	if let strongSelf = self {
		/// ....
	}
};
```

这种写法的缺点在于，我们不能写 `if let self = self`，因此需要重新定义一个变量 `strongSelf`，命名方式显得不够优雅。

除此以外还可以使用 Swift 标准库提供的函数 `withExtendedLifetime `：

```swift
self.completionHandler = { [weak self] in
	withExtendedLifetime(self) {
		/// ...
	}
};
```

这种写法的缺点在于，`self` 依然是可选类型的，还需要把它解封后才能使用。

最后，还有一种解决方案是自定义 `withExtendedLifetime `函数：

```swift
extension Optional {
	func withExtendedLifetime(body: T -> Void) {
		if let strongSelf = self {
			body(strongSelf)
		}
	}
}
```

至于这种写法是否更加优雅，就见仁见智了：

```swift
self.completionHandler = { [weak self] in
	self.withExtendedLifetime {
		/// 这里用 $0 表示 self
	}
};
```

# 参考资料

1. [What happens when a block is set to nil during its execution?](http://stackoverflow.com/questions/12272783/what-happens-when-a-block-is-set-to-nil-during-its-execution)
2. [The Weak/Strong Dance in Swift](http://kelan.io/2015/the-weak-strong-dance-in-swift/)