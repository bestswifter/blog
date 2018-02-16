从最初开始学习 iOS 的时候，我们就被告知 UI 操作一定要放在主线程进行。这是因为 UIKit 的方法不是线程安全的，保证线程安全需要极大的开销。那么问题来了，在主线程中进行 UI 操作一定是安全的么？

> 显然，答案是否定的!

在苹果的 `MapKit` 框架中，有一个叫做 `addOverlay` 的方法，它在底层实现的时候，不仅仅要求代码执行在主线程上，还要求执行在 GCD 的主队列上。这是一个极罕见的问题，但已经有人在使用 ReactiveCocoa 时踩到了坑，并提交了 [issue](https://github.com/ReactiveCocoa/ReactiveCocoa/issues/2635#issuecomment-170215083)。

苹果的 Developer Technology Support 承认这是一个 bug。不管这是 bug 还是历史遗留设计，也不管是不是在钻牛角尖，为了避免再次掉进同样的坑，我认为都有必要分析一下问题发生的原因和解决方案。

## GCD 知识复习

在 GCD 中，使用 `dispatch_get_main_queue()` 函数可以获取主队列。调用 `dispatch_sync()` 方法会把任务同步提交到指定的队列。

注意一下队列和线程的区别，他们之间并没有“拥有关系(ownership)”，当我们同步的提交一个任务时，首先会阻塞当前队列，然后等到下一次 runloop 时再在合适的线程中执行 block。

在执行 block 之前，首先会寻找合适的线程来执行block，然后阻塞这个线程，直到 block 执行完毕。寻找线程的规则是: **任何提交到主队列的 block 都会在主线程中执行**，在不违背此规则的前提下，文档还告诉我们系统会自动进行优化，**尽可能的在当前线程执行 block**。

顺便补充一句，GCD 死锁的充分条件是:“向当前队列重复同步提交 block”。从原理来看，死锁的原因是提交的 block 阻塞了队列，而队列阻塞后永远无法执行完 `dispatch_sync()`，可见这里完全和代码所在的线程无关。

另一个例子也可以证明这一点，在主线程中向一个串行队列同步的派发 block，根据上文选择线程的原则，block 将在主线程中执行，但同样不会导致死锁:

```objc
dispatch_queue_t queue = dispatch_queue_create("com.kt.deadlock", nil);
dispatch_sync(queue, ^{
    NSLog(@"current thread = %@", [NSThread currentThread]);
});
// 输出结果:
// current thread = <NSThread: 0x7fa7fb403e90>{number = 1, name = main} 
```

## 原因分析

啰嗦了这么多，回到之前描述的 bug 中来。现在我们知道，即使是在主线程中执行的代码，也很可能不是运行在主队列中(反之则必然)。如果我们在子队列中调用  `MapKit` 的 `addOverlay` 方法，即使当前处于主线程，也会导致 bug 的产生，因为这个方法的底层实现判断的是主队列而非主线程。

更进一步的思考，有时候为了保证 UI 操作在主线程运行，如果有一个函数可以用来创建新的 `UILabel`，为了确保线程安全，代码可能是这样:

```objc
- (UILabel *)labelWithText: (NSString *)text {
    __block UILabel *theLabel;
    if ([NSThread isMainThread]) {
        theLabel = [[UILabel alloc] init];
        [theLabel setText:text];
    }
    else {
        dispatch_sync(dispatch_get_main_queue(), ^{
            theLabel = [[UILabel alloc] init];
            [theLabel setText:text];
        });
    }
    return theLabel;
}
```

从严格意义上来讲，这样的写法不是 100% 安全的，因为我们无法得知相关的系统方法是否存在上述 Bug。

## 解决方案

由于提交到主队列的 block 一定在主线程运行，并且在 GCD 中线程切换通常都是由指定某个队列引起的，我们可以做一个更加严格的判断，即用判断是否处于主队列来代替是否处于主线程。

GCD 没有提供 API 来进行相应的判断，但我们可以另辟蹊径，利用 `dispatch_queue_set_specific` 和 `dispatch_get_specific` 这一组方法为主队列打上标记:

```objc
+ (BOOL)isMainQueue {
    static const void* mainQueueKey = @"mainQueue";
    static void* mainQueueContext = @"mainQueue";
    
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        dispatch_queue_set_specific(dispatch_get_main_queue(), mainQueueKey, mainQueueContext, nil);
    });
    
    return dispatch_get_specific(mainQueueKey) == mainQueueContext;
}
```

用 `isMainQueue` 方法代替 `[NSThread isMainThread]` 即可获得更好的安全性。

## 参考资料

1. [Community bug reports about MapKit](http://www.openradar.me/24025596)
2. [GCD's Main Queue vs Main Thread](http://blog.benjamin-encz.de/post/main-queue-vs-main-thread/)
3. [ReactiveCocoa 中遇到类似的坑](https://github.com/ReactiveCocoa/ReactiveCocoa/issues/2635#issuecomment-170215083)
4. [Why can't we use a dispatch_sync on the current queue?](http://stackoverflow.com/questions/10984732/why-cant-we-use-a-dispatch-sync-on-the-current-queue)