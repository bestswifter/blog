在讨论 runloop 相关的文章，以及分析 AFNetworking(2.x) 源码的文章中，我们经常会看到关于利用 runloop 进行线程保活的分析，但如果不求甚解的话，极有可能因此学会了一个错误的用法，本文就来分析一下其中常见的误区。

我提供了一个 Demo，可以在我的 [Github](https://github.com/bestswifter/MySampleCode/tree/master/RunloopAndThread) 上下载并运行一遍，文章中只提供了部分代码。

## AFN 中的实现

首先我们知道在[旧版本的AFN](https://github.com/AFNetworking/AFNetworking/tree/2.x) 中使用了 NSURLConnection 来发起并处理网络连接。AFN 的做法是把网络请求的发起和解析都放在同一个子线程中进行，但由于子线程默认不开启 runloop，它会向一个 C语言程序那样在运行完所有代码后退出线程。而网络请求是异步的，这会导致获取到请求数据时，线程已经退出，代理方法没有机会执行。因此，AFN 的做法是使用一个 runloop 来保证线程不死，也就是下面这段被讲烂了的代码:

```objc
+ (void)networkRequestThreadEntryPoint:(id)__unused object {
    @autoreleasepool {
        [[NSThread currentThread] setName:@"AFNetworking"];

        NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
        [runLoop addPort:[NSMachPort port] forMode:NSDefaultRunLoopMode];
        [runLoop run];
    }
}
```

当然，单独看这一个方法意义不大，我们稍微结合一下上下文，看看这个方法在哪里被调用:

```objc
+ (NSThread *)networkRequestThread {
    static NSThread *_networkRequestThread = nil;
    static dispatch_once_t oncePredicate;
    dispatch_once(&oncePredicate, ^{
        _networkRequestThread = [[NSThread alloc] initWithTarget:self selector:@selector(networkRequestThreadEntryPoint:) object:nil];
        [_networkRequestThread start];
    });

    return _networkRequestThread;
}
```

似乎这种写法提供了一种思路:“如果需要在子线程中异步执行操作，可以利用 runloop 进行线程保活”。但准确的来说，AFN 的这种写法并不能实现我们的需求，它只是在 AFN 这个特殊场景下可以工作。

不信你可以尝试阅读一下第二段代码，看看它和平时使用 `NSThread` 时有什么区别，如果没看出来也无妨，先记住这段代码，我们稍后分析。

## NSThread 与内存泄漏

这种写法的第一个问题就是存在内存泄漏。我们构造以下用例，其实就是把 AFN 的线程创建放在一个循环里:

```objc
- (void)memoryTest {
    for (int i = 0; i < 100000; ++i) {
        NSThread *thread = [[NSThread alloc] initWithTarget:self selector:@selector(run) object:nil];
        [thread start];
    }
}

- (void)run {
    @autoreleasepool {
        NSLog(@"current thread = %@", [NSThread currentThread]);
        NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
        if (!self.emptyPort) {
            self.emptyPort = [NSMachPort port];
        }
        [runLoop addPort:self.emptyPort forMode:NSDefaultRunLoopMode];
        [runLoop run];
    }
}
```

奇怪的事情出现了，尽管是在 ARC 环境下，内存依然不停的上涨。如果我们把 `run` 方法中和 runloop 相关的代码删除则不会出现上述问题，显然，开启 runloop 导致了内存泄漏，也就是 `thread` 对象无法释放。

> 这里的 emptyPort 用来维持 runloop 的运行，根据官方文档的描述，如果 runloop 中没有任何 modeItem，就不会启动，而是立刻退出。之所以选择作为属性而不是临时变量，是因为我发现每次调用 [NSMachPort port] 方法都会占用内存，原因暂时不清楚。

我们可以尝试手动结束 runloop 并关闭线程:

```objc
- (void)memoryTest {
    for (int i = 0; i < 100000; ++i) {
        NSThread *thread = [[NSThread alloc] initWithTarget:self selector:@selector(run) object:nil];
        [thread start];
        [self performSelector:@selector(stopThread) onThread:thread withObject:nil waitUntilDone:YES];
    }
}

- (void)stopThread {
    CFRunLoopStop(CFRunLoopGetCurrent());
    NSThread *thread = [NSThread currentThread];
    [thread cancel];
}
```

很遗憾，这依然没有任何效果。而且不难猜测是我们没有能正确的结束 runloop 的运行。

## Runloop 的启动与退出

考验英文水平的时候到了，首先来看一段[官方文档](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html#//apple_ref/doc/uid/10000057i-CH16-SW1)对于如何启动 runloop 的介绍，它的启动方式一共有三种:

1. Unconditionally
2. With a set time limit
3. In a particular mode

这三种进入方式分别对应了三种方法，其中第一种就是我们目前使用的:

1. run
2. runUntilDate
3. runMode:beforeDate:

接下来分别是对三种方式的介绍，文字比较啰嗦，这里我简单总结一下，有兴趣的读者可以直接看原文。

* 无条件进入是最简单的做法，但也最不推荐。这会使线程进入死循环，从而不利于控制 runloop，结束 runloop 的唯一方式是 kill 它。
* 如果我们设置了超时时间，那么 runloop 会在处理完事件或超时后结束，此时我们可以选择重新开启 runloop。这种方式要优于前一种
* 这是相对来说最优秀的方式，相比于第二种启动方式，我们可以指定 runloop 以哪种模式运行。

查看 `run` 方法的文档还可以知道，它的本质就是无限调用 `runMode:beforeDate:` 方法，同样地，`runUntilDate:` 也会重复调用 `runMode:beforeDate:`，区别在于它超时后就不会再调用。

**总结来说，`runMode:beforeDate:` 表示的是 runloop 的单次调用，另外两者则是循环调用。**

相比于 runloop 的启动，它的退出就比较简单了，只有两种方法:

1. 设置超时时间
2. 手动结束

如果你使用方法二或三来启动 runloop，那么在启动的时候就可以设置超时时间。然而考虑到目标是:“利用 runloop 进行线程保活”，所以我们希望对线程和它的 runloop 有最精确的控制，比如在完成任务后立刻结束，而不是依赖于超时机制。

好在根据文档的描述，我们还可以使用 `CFRunLoopStop()` 方法来手动结束一个 runloop。注意文档中在介绍利用 `CFRunLoopStop()` 手动退出时有下面这句话:

> The difference is that you can use this technique on run loops you started unconditionally.

这里的解释非常容易产生误会，如果在阅读时没有注意到 **exit** 和 **terminate** 的微小差异就很容易掉进坑里，因为在 `run` 方法的文档中还有这句话:

> If you want the run loop to terminate, you shouldn't use this method

**总的来说，如果你还想从 runloop 里面退出来，就不能用 `run` 方法。根据实践结果和文档，另外两种启动方法也无法手动退出。**

## 正确的做法

难道子线程中开启了 runloop 就无法结束并释放了么？这显然是一个不合理的结论，经过一番查找，终于在[这篇文章](http://www.cocoabuilder.com/archive/cocoa/305204-runloop-not-being-stopped-by-cfrunloopstop.html)里找到了答案，它给出了使用 `CFRunLoopStop()` 无效的原因:

> CFRunLoopStop() 方法只会结束当前的 runMode:beforeDate: 调用，而不会结束后续的调用。

这也就是为什么 Runloop 的文档中说 `CFRunLoopStop()` 可以 **exit(退出)** 一个 runloop，而在 `run` 等方法的文档中又说这样会导致 runloop 无法 **terminate(终结)**。

文章中给出的方案是使用 `CFRunLoopRun()` 启动 runloop，这样就可以通过 `CFRunLoopStop()` 方法结束。而文档则推荐了另一种方法:

```objc
BOOL shouldKeepRunning = YES;        // global
NSRunLoop *theRL = [NSRunLoop currentRunLoop];
while (shouldKeepRunning && [theRL runMode:NSDefaultRunLoopMode beforeDate:[NSDate distantFuture]]);
```

我尝试了文档提供的方法，确实不会导致内存泄漏，但不方便验证 runloop 是否真的开启，然后又被终止。所以我实际采用的是第一种方案:

```objc
- (void)memoryTest {
    for (int i = 0; i < 100000; ++i) {
        NSThread *thread = [[NSThread alloc] initWithTarget:self selector:@selector(run) object:nil];
        [thread start];
        [self performSelector:@selector(stopThread) onThread:thread withObject:nil waitUntilDone:YES];
    }
}

- (void)stopThread {
    CFRunLoopStop(CFRunLoopGetCurrent());
    NSThread *thread = [NSThread currentThread];
    [thread cancel];
}

- (void)run {
    @autoreleasepool {
        NSLog(@"current thread = %@", [NSThread currentThread]);
        NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
        if (!self.emptyPort) {
            self.emptyPort = [NSMachPort port];
        }
        [runLoop addPort:self.emptyPort forMode:NSDefaultRunLoopMode];
        [runLoop runMode:NSRunLoopCommonModes beforeDate:[NSDate distantFuture]];
    }
}
```

## 验证

采用上述方案后，确实可以观察到不会再出现内存泄漏问题，但这并不是终点。因为我们还需要验证 runloop 确实在启动后被关闭。

为了证明 runloop 确实启动，我设计了如下方法:

```objc
- (void)printSomething {
    NSLog(@"current thread = %@", [NSThread currentThread]);
    [self performSelector:@selector(printSomething) withObject:nil afterDelay:1];
}
```

我们知道 `performSelector:withObject:afterDelay` 依赖于线程的 runloop，因为它本质上是由一个定时器负责定期加入到 runloop 中执行。所以如果这个方法可以成功执行，说明当前线程的 runloop 已经开启，否则则说明没有启动。

为了证明 runloop 可以被终止，我创建了一个按钮，在点击按钮时执行以下方法:

```objc
- (void)stopButtonDidClicked:(id)sender {
    [self performSelector:@selector(stopRunloop) onThread:self.thread withObject:nil waitUntilDone:YES];
}

- (void)stopRunloop {
    CFRunLoopStop(CFRunLoopGetCurrent());
}
```

成功的观察到点击按钮后，控制台不再有日志输出，因此证明 runloop 确实已经停止。

## 总结

啰嗦了这么多，其实是为了研究如何利用 runloop 实现线程保活。要注意的地方主要有以下点:

1. 了解 runloop 实现线程保活的原理，注意添加的那个空 port
2. 了解 runloop 导致的线程对象内存泄漏问题
3. 了解 runloop 的几种启动方式以及彼此之间的关联
4. 了解 runloop 的释放方式和原理

由于相关资料的匮乏以及个人水平有限，虽然竭力研究但仍不保证绝对的正确性，欢迎交流指正。

最后，文章开头对 AFN 的分析留作一个简单的思考题，为什么 AFN 中的用法不会有问题？

### 参考资料

1. [Run Loops 官方文档](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html)
2. [Runloop not being stopped by CFRunLoopStop?](http://www.cocoabuilder.com/archive/cocoa/305204-runloop-not-being-stopped-by-cfrunloopstop.html)
3. [深入理解 RunLoop](http://blog.ibireme.com/2015/05/18/runloop/)