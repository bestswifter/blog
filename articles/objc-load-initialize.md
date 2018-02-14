OC 中有两个特殊的类方法，分别是 `load` 和 `initialize`。本文总结一下这两个方法的区别于联系、使用场景和注意事项。Demo 可以在我的 Github 上找到——[load 和 initialize](https://github.com/bestswifter/MySampleCode/tree/master/load)，如果觉得有帮助还望点个star以示支持，总结在文章末尾。

# load

顾名思义，`load`方法在这个文件被程序装载时调用。只要是在Compile Sources中出现的文件总是会被装载，这与这个类是否被用到无关，因此`load`方法总是在`main`函数之前调用。

### 调用规则
 
如果一个类实现了`load`方法，在调用这个方法前会首先调用父类的`load`方法。而且这个过程是自动完成的，并不需要我们手动实现：

```objc
// In Parent.m
+ (void)load {
    NSLog(@"Load Class Parent");
}

// In Child.m，继承自Parent
+ (void)load {
    NSLog(@"Load Class Child");
}

// In Child+load.m，Child类的分类
+ (void)load {
    NSLog(@"Load Class Child+load");
}

// 运行结果：
/*
2016-02-01 21:28:14.379 load[11789:1435378] Load Class Parent
2016-02-01 21:28:14.380 load[11789:1435378] Load Class Child
2016-02-01 22:28:14.381 load[11789:1435378] Load Class Child+load
*/
```

如果一个类没有实现`load`方法，那么就不会调用它父类的`load`方法，这一点与正常的类继承和方法调用不一样，需要额外注意一下。

### 执行顺序

`load`方法调用时，系统处于脆弱状态，如果调用别的类的方法，且该方法依赖于那个类的`load`方法进行初始化设置，那么必须确保那个类的`load`方法已经调用了，比如demo中的这段代码，打印出的字符串就为`null`：

```objc
// In Child.m
+ (void)load {
    NSLog(@"Load Class Child");
    
    Other *other = [Other new];
    [other originalFunc];
    
    // 如果不先调用other的load，下面这行代码就无效，打印出null
    [Other printName];
}
```

`load`方法的调用顺序其实有迹可循，我们看到demo的项目设置如下：

![执行顺序](http://images.bestswifter.com/load/order.png)

在Compile Sources中，文件的排放顺序就是其装载顺序，自然也就是`load`方法调用的顺序。这一点也证明了`load`方法中会自动调用父类的方法，因为在demo的输出结果中，`Parent`的`load`方法先于`Child`调用，而它的装载顺序其实在`Child`之后。

虽然在这种简单情况下我们可以辨别出各个类的`load`方法调用的顺序，但**永远不要**依赖这个顺序完成你的代码逻辑。一方面，这在后期的开发中极容易导致错误，另一方面，你实际上并不需要这么做。

### 使用场景

由于调用`load`方法时的环境很不安全，我们应该尽量减少`load`方法的逻辑。另一个原因是`load`方法是线程安全的，它内部使用了锁，所以我们应该避免线程阻塞在`load`方法中。

一个常见的使用场景是在`load`方法中实现Method Swizzle：

```objc
// In Other.m
+ (void)load {
    Method originalFunc = class_getInstanceMethod([self class], @selector(originalFunc));
    Method swizzledFunc = class_getInstanceMethod([self class], @selector(swizzledFunc));
    
    method_exchangeImplementations(originalFunc, swizzledFunc);
}
```

在`Child`类的`load`方法中，由于还没调用`Other`的`load`方法，所以输出结果是"Original Output"，而在main函数中，输出结果自然就变成了"Swizzled Output"。

一般来说，除了Method Swizzle，别的逻辑都不应该放在`load`方法中实现。

# initialize

这个方法在第一次给某个类发送消息时调用（比如实例化一个对象），并且只会调用一次。`initialize`方法实际上是一种惰性调用，也就是说如果一个类一直没被用到，那它的`initialize`方法也不会被调用，这一点有利于节约资源。

### 调用规则

与`load`方法类似的是，在`initialize`方法内部也会调用父类的方法，而且不需要我们显示的写出来。与`load`方法不同之处在于，即使子类没有实现`initialize`方法，也会调用父类的方法，这会导致一个很严重的问题：

```objc
// In Parent.m
+ (void)initialize {
    NSLog(@"Initialize Parent, caller Class %@", [self class]);
}

// In Child.m
// 注释掉initialize方法

// In main.m
Child *child = [Child new];
```

运行后发现父类的`initialize`方法竟然调用了两次：

```objc
2016-02-01 22:57:02.985 load[12772:1509345] Initialize Parent, caller Class Parent
2016-02-01 22:57:02.985 load[12772:1509345] Initialize Parent, caller Class Child
```

这是因为在创建子类对象时，首先要创建父类对象，所以会调用一次父类的`initialize`方法，然后创建子类时，尽管自己没有实现`initialize`方法，但还是会调用到父类的方法。

虽然`initialize`方法对一个类而言只会调用一次，但这里由于出现了两个类，所以调用两次符合规则，但不符合我们的需求。正确使用`initialize`方法的姿势如下：

```objc
// In Parent.m
+ (void)initialize {
    if (self == [Parent class]) {
        NSLog(@"Initialize Parent, caller Class %@", [self class]);
    }
}
```

加上判断后，就不会因为子类而调用到自己的`initialize`方法了。

### 使用场景

`initialize`方法主要用来对一些不方便在编译期初始化的对象进行赋值。比如`NSMutableArray`这种类型的实例化依赖于runtime的消息发送，所以显然无法在编译器初始化：

```objc
// In Parent.m
static int someNumber = 0;	 // int类型可以在编译期赋值
static NSMutableArray *someObjects;

+ (void)initialize {
    if (self == [Parent class]) {
	    // 不方便编译期复制的对象在这里赋值
        someObjects = [[NSMutableArray alloc] init];
    }
}
```

# 总结

1. `load`和`initialize`方法都会在实例化对象之前调用，以main函数为分水岭，前者在main函数之前调用，后者在之后调用。这两个方法会被自动调用，不能手动调用它们。
2. `load`和`initialize`方法都不用显示的调用父类的方法而是自动调用，即使子类没有`initialize`方法也会调用父类的方法，而`load`方法则不会调用父类。
3. `load`方法通常用来进行Method Swizzle，`initialize`方法一般用于初始化全局变量或静态变量。
4. `load`和`initialize`方法内部使用了锁，因此它们是线程安全的。实现时要尽可能保持简单，避免阻塞线程，不要再使用锁。