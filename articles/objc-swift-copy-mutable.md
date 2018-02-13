# Objective-C

为了解释方便，定义两个类：`Person`和`MyObject`，它们都继承自`NSObject`。他们的关系如下:

```objc
// Person.h
@property (strong, nonatomic, nullable) MyObject *object;
```

```objc
// MyObjec.h
@property (copy, nonatomic) NSString *name;
```

## 普通对象拷贝

对于一个OC中的对象来说，可能涉及拷贝的有三种操作：

1. `retain`操作：

	```objc
	Person *p = [[Person alloc] init];
	Person *p1 = p;
	```
	
	这里的`p1`默认是`__strong`，所以它会对`p`进行`retain`操作。`retain`与复制无关，只会对引用计数加1。`p1`和`p`的地址是完全一样的：
	
	```objc
	2015-12-23 21:24:31.893 Copy[1300:120857] p = 0x1006012c0
2015-12-23 21:24:31.894 Copy[1300:120857] p1 = 0x1006012c0
	```
	
	这种写法最简单，而且严格来说不是复制，但值得一提，因为在接下来的OC和Swift中，都会涉及到这样的代码。
	
2. `copy`方法：

	调用`copy`方法需要实现`NSCopying`协议，并提供`copyWithZone`方法：
	
	```objc
	- (id)copyWithZone:(NSZone *)zone {
    	Person *copyInstance = [[self class] allocWithZone:zone];
    	copyInstance.object = self.object;
    	return copyInstance;
	}
	```
	
	第二行代码就是刚刚所说的`retain`操作。因此，我们虽然复制了`Person`对象的指针，但是其内部的属性，依然和原来对象的相同。
	
3. 自定义拷贝方法：
	
	我们当然可以自己定义一个拷贝方法，在复制`Person`对象的同时，把其中的`object`属性也复制，而不是仅仅`retain`。
	

第二三种复制方法的区别如图所示：

![两种拷贝方式](http://upload-images.jianshu.io/upload_images/1171077-c54f3506af03483b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 浅拷贝与深拷贝

标为红色的是两种拷贝方式的不同之处。对于左边这种，只拷贝指针本身的拷贝方法，我们称为浅拷贝。对于右边那种，不仅拷贝指针自身，还拷贝指针中所有元素的拷贝方法，我们称为深拷贝。

没有明确的限制`copy`和自定义的拷贝方法要如何实现。也就是说`copy`方法可以用来进行深拷贝，我们也可以自定义浅拷贝的方法。这完全取决于我们自己如何实现`copy`方法和自定义的拷贝方法。在OC中，对于自定义的类来说，浅拷贝与深拷贝只是一种概念，并没有明确的标注哪种方法就是浅拷贝。

**注意**

“深拷贝将被拷贝的对象完全复制”这种说法不完全正确。比如上图中我们看到`data`的地址永远不会拷贝。这是因为，深拷贝只负责了对象的拷贝，以及对象中所有属性的拷贝。正是因为拷贝了属性，将`p`深拷贝后得到的`p'`的`object`指针地址和`p`的`object`指针地址不同。

但是至于`data`会不会被拷贝，这取决于`MyObject`类如何设计，如果`MyObject`的`copy`方法只是浅拷贝，就会形成如上图所示的情况。如果`MyObject`的`copy`方法也是深拷贝，那么`data`的地址也会不同。

## 容器对象拷贝

在OC中，所有`Foundation`中的容器类，分为可变容器和不可变容器，它们的拷贝**都是浅拷贝**。这也就是为什么建议自定义的对象实现浅拷贝，如果有需要才自定义深拷贝方法。因为这样一来，所有的方法调用就都可以统一，不至于造成误解。

如果我们把数组想象成一个三层的数据结构，第一层是数组自己的指针，第二层是存放在数组中的指针，第三层(如果第二层是指针)则是这些指针指向的对象。那么在复制数组时，复制的是前两层，第三层的对象不会被复制。如果把前两层看做指针，第三层看做对象，那么数组的拷贝，无论是`copy`还是`mutableCopy`都是浅拷贝。当然，也有人把这个称为“单层深拷贝”，这些概念性的定义都不重要，重要的是知道数组拷贝时的原理。

这一点很好理解。首先，指针所指向的对象，也许很大，深拷贝可能占用过多的内存和时间。其次，容器不知道自己存储的对象是否实现了`NSCopying`协议。如果容器的拷贝默认是深拷贝，同时你在数组中存放了`Person`类的对象，而`Person`类根本没有实现`NSCopying`协议，后果是复制容器会导致程序崩溃。这是任何语言开发者都不希望看到的，所以设身处地想一下，如果是你来设计OC，也不会让数组深拷贝吧。

观察下面这段代码，思考一下为什么`a1[0] = @0`没有影响`a2`:

```objc
NSMutableArray *a1 = [[NSMutableArray alloc] initWithObjects:@1, @2, nil];
NSMutableArray *a2 = [a1 mutableCopy];
a1[0] = @0;
NSLog(@"a2 = %@", a2);

/*
2015-12-23 23:11:53.711 Copy[1795:220469] a2 = (
    1,
    2
)
*/
```

## 可变性

容器对象分为可变容器与不可变容器，`NSData`、`NSArray`、`NSString`等都是不可变容器，以`NSMutable`开头的则是它们的可变版本。下面统一用`NSArray`和`NSMutableArray`举例说明。

因为`NSMutableArray`是`NSArray`的子类，所以`NSArray`对象不能强制转换成`NSMutableArray`，否则在调用`addObject`方法时会崩溃。反之，`NSMutableArray`可以转换成它的父类`NSArray`，这么做会导致它失去可变性。

容器拷贝的难点在于可变性的变化。容器有两种方法：`copy`和`mutableCopy`，再次强调这两者都是浅拷贝。它们的区别在于，返回值是否是可变的。前者返回不可变容器，后者返回可变容器。

这也就是说，返回值的可变性与被拷贝对象的可变性无关，仅取决于调用了何种拷贝方法。比如：

```objc
NSMutableArray *mutableArray = [[NSMutableArray alloc] initWithObjects:@1, @2, nil];
NSMutableArray *array = [mutableArray copy];
[array addObject:@3];
```

尽管我们调用了`mutableArray`的拷贝方法，返回值也声明为`NSMutableArray`，但是调用`addObject`方法时依然会导致运行时错误。这是由错误的调用了`copy`方法导致的。

调用一个对象的浅拷贝方法会得到一个新的对象(地址不同)，但是容器类中有一个特例：

```objc
NSArray *array1 = @[@1, @2];
NSArray *array2 = [array1 copy];
// array1和array2地址相同
```

这是因为既然`array1`和`array2`都不能再修改，那么两者共用同一块内存也是无所谓的，所以OC做了这样的优化。

## 字符串拷贝

字符串也可以被当做容器来理解。它有`NSString`和`NSMutableString`两个版本。

于是为什么字符串属性要定义成`@property(copy, nonatomic)`就很好理解了。它主要用于处理这种情况：

```objc
NSMutableString *string = @"hello";
self.propertyString = string;
[string appendString:@" world"];
```

如果属性定义成`strong`，那么在第二步执行了`retain`操作，第三步对`string`的修改就会影响到原来的属性。现在我们把属性定义为`copy`，那么第二步操作其实是得到了一个新的，不可变字符串。这符合我们的预期目的。

# Swift拷贝

## 结构体拷贝

数组、字典等容器在Swift中被定义成了结构体，它们的拷贝规则和OC完全不同：

```swift
var array1 = [1,2,3]
var array2 = array1

array1[0] = 0
print(array2) // 输出结果：[1, 2, 3]
```

可以看到，即使是最简单的等号赋值，也会浅拷贝原来的值。这是由Swift中结构体的值语义决定的。之所以说是浅拷贝而不是深拷贝，理由参见前文解释OC中容器的浅拷贝，尤其是第二点理由，不管是对于OC还是Swift来说都是通用的。

## 对象拷贝

和OC中指针赋值类似，对象的直接赋值操作与拷贝无关：

```swift
class Person {
    var name: String;
    init(name:String) {
        self.name = name
    }
}

let person1 = Person(name: "zxy")
let person2 = person1
person1.name = "new name"

print(person2.name) //结果是“new name”
```

如果要拷贝对象，有两种方法。首先，最自然想到的是实现`NSCopying`协议，注意只有`NSObject`类的对象才能实现这个协议：

```swift
class Person : NSObject, NSCopying {
    var name: String;
    init(name:String) {
        self.name = name
    }

    func copyWithZone(zone: NSZone) -> AnyObject {
        return Person(name: self.name)
    }
}

```

但这样做最大的问题在于，你必须继承自`NSObject`，这就又回到了OC的那一套。如果我们希望定义纯粹的Swift类，完全可以自己定义并实现拷贝方法。

“面向接口编程”的原则告诉我们，我们应该让`Person`实现某个接口，而不是继承自某个子类：

```swift
protocol Copyable {
    func copy() -> Copyable
}

class Person : Copyable {
    var name: String;
    init(name:String) {
        self.name = name
    }

    func copy() -> Copyable {
        return Person(name: self.name)
    }
}

let person1 = Person(name: "zxy")
let person2 = person1.copy() as! Person
```

这样就完美的实现Swift-Style拷贝了。

# 总结

在OC中，浅拷贝通常由`NSCopying`协议的`copyWithZone`方法实现，深拷贝需要自定义方法。直接复制意味着`retain`而不是拷贝。

在Swift中，值类型直接用等号赋值意味着浅拷贝，引用类型的拷贝可以通过实现自定义的`Copyable`协议或OC的`NSCopying`协议完成。

在OC中，我们需要容器的可变性，而Swift在这一点做的要比OC好得多。它的可变性非常简单，完全通过`let`和`var`控制，这也是Swift相比于OC的一个优点吧，毕竟高级的语言应该尽可能封装底层实现。