> 仅是一家之言，欢迎交流讨论、指正错误。

本科有一门课程是设计模式，上课的时候读完了《Head First 设计模式》，这是一本很好的书，可惜当时的我不是一个好读者，囫囵吞枣看了几百页却没有吸收精华。工作以后，有了一些代码量的积累，打算补一补设计模式相关的内容。

这篇文章讲工厂模式，在开始分析之前，我想谈谈我对设计模式的看法。以前我之所以学不好设计模式，一方面是自己代码量不足，只能纸上谈兵，另一方面我想也和介绍设计模式的文章有关。大多数文章重点在于介绍 xxx 模式是什么，然后配上千篇一律的类图(Class Diagram) 和 Demo。好一点的文章，类图和 Demo 容易理解点，有可能还会谈谈某两个模式之间的异同。

但设计模式是什么？是一门必须掌握的课程，一些必须背下来的概念，然后放到实际工程里面套用的么？我不敢苟同，在我看来涉及模式其实描述了一些 Coding 的技巧。所谓的技巧，有的是能够节省冗余代码，更重要的则是开闭原则，也就是“**对拓展开放，对修改关闭**”，或者说得再直白点，就是方便开发者后期维护的。

这些技巧是有限的、反复出现的，为了便于交流和沟通，我们给这些技巧起上名字，否则每个人对这些技巧都有自己的理解，就不方便沟通了。既然是技巧，那么一定有它**“巧”**的一面，对比的则是原来不巧的代码。所以理解某个设计模式的实现是次要的，重点是理解它巧在哪里，因为要理解巧在哪里，所以顺便要看看如何实现。

# 工厂模式

上面这些话可能有点虚、有点绕，没关系，我举个具体例子来说。这篇文章介绍的是工厂模式，工厂模式根据“教科书”，分为三种:

1. 简单工厂模式
2. 工厂方法模式
3. 抽象工厂模式

既然是工厂，那么肯定是用来生产东西的，所以有的教材或者书籍把它归类于“创建型模式”，我是坚决反对的。还有很多文章，动不动就是工厂、材料的举例，如果你以为只有创建东西，还需要材料时才用得着工厂模式，那么设计模式这门学问基本上就算失败了。

## 简单工厂模式

以那个经典的披萨的例子来说吧，父类叫 `Pizza`，子类有很多，什么 `GoodPizza`、`BadPizza`、`LargePizza`、`SmallPizza` 之类的随便写。

问题来了，这么多可能的披萨，怎么选择呢？当然是用参数来标记了。比如订披萨的时候:

```java
// Snippet 1
public Pizza orderPizza(String type){
    Pizza pizza = null;
    if(type.equals("g")){
        pizza = new GoodPizza();
    }else if(type.equals("b")){
        pizza = new BadPizza();
    }else if(type.equals("l")){
        pizza = new LargePizza();
    }else {
        pizza = new SmallPizza();
    }
    return pizza;
 }
```

第一段代码是原始场景，它只是实现了需求，我给它起个名字叫学校代码(School Code)，也就是那些学校里的学生写出来的，仅仅是可以运行的代码。问题很明显，一方面你不能保证只有在订披萨的时候才会创建披萨的实例对象，如果别的地方也要创建披萨对象，相同的代码就要重写一遍。另一方面这样写会导致 `orderPizza` 所在的类依赖于 `Pizza` 的四个子类，而实际上的需求仅仅是创建披萨实例而已，它的调用者并不应该依赖于具体的子类。

因此在订披萨的地方处理这样的字符串判断就显得不合理，解决方案也很简单，用一个专门的类来处理创建披萨的逻辑就行了:

```java
//Snippet 2
public class SimplePizzaFactory {
    public Pizza createPizza(String type) {
        Pizza pizza = null;
        if(type.equals("g")){
            pizza = new GoodPizza();
        }else if(type.equals("b")){
            pizza = new BadPizza();
        }else if(type.equals("l")){
            pizza = new LargePizza();
        }else {
            pizza = new SmallPizza();
        }
        return pizza;
    }
}

public Pizza orderPizza(String type) {
    SimplePizzaFactory simplePizzaFactory = new SimplePizzaFactory();
    Pizza pizza= simplePizzaFactory.createPizza(type);
}
```

这就是简单工厂方法了，它说的是两个概念:

1. 一个类只做和自己相关的事，不依赖的别瞎依赖
2. 有可能复用的代码抽出去，独立成类，不要到处重复

这个模式和什么所谓的 **工厂** 一点关系都没有。假如这里调用的不是 `new` 关键字来新建对象，而是用类的静态方法，一样会有依赖问题。这种大段的细节逻辑，不管是不是创建对象，还是可以被独立出去。

采用了简单工厂模式以后，n 个披萨需要 n+1 个类，额外的那个是工厂类，处理创建披萨对象的具体逻辑。

## 工厂方法模式

我们考虑一下新增披萨种类的情况。在第二段代码的实现中，如果要新增一个子类，需要在 `SimplePizzaFactory` 中新增对子类的依赖和解析方法，也就是多一个 `else if` 的分支。如果是删除一种披萨，就需要删掉一个子类的依赖和 `else if` 分支。

这个操作看起来并不复杂，虽然要增改代码，但还是可以接受。不过如果子类无法修改 `SimplePizzaFactory` 代码呢？父类和工厂类有可能是基础团队在维护，而披萨子类可能是业务团队维护，不同的业务团队还有可能维护不同的子类。且不说不一定有代码的写权限，就算大家一起写，代码冲突了怎么办，忘记删除逻辑了怎么办？

这时候就看出问题所在了，它违反了开闭原则，也就是说并没有做到对修改关闭。怎么对修改关闭呢，这就需要借助 OOP 编程时的一个小技巧。

直接上代码吧，为了偷懒，我直接把 [深入浅出工厂设计模式](https://segmentfault.com/p/1210000009074890/read) 这篇文章的里的代码搬过来改改了:

```java
//snippet 3
public abstract class APizzaStore {
    public Pizza orderPizza(String type) {
        Pizza pizza= createPizza(type);
        return pizza;
    }
    abstract Pizza createPizza(String type);
}

public class GoodPizzaStore extends APizzaStore{
    @Override
    Pizza createPizza(String type) {
        Pizza pizza = new GoodPizza();
        return pizza;
    }
}
```

采用工厂方法模式以后，工厂不再负责具体的业务细节。它变成了一个抽象类，规定了一个**抽象方法** `createPizza` **强迫** 子类实现。同时它在 `orderPizza` 函数中调用了这个方法，但方法的实现者并不是自己。因为实际使用的并不是抽象工厂类 `APizzaStore` 而是具体的 `GoodPizzaStore`，利用**多态性**，实际调用的也是子类的 `createPizza` 方法。

所以归根结底，抽象工厂方法只是使用了一个小技巧，我称之为:

> 父类定框架，子类做填充，依赖多态性

这种技巧的好处在于将具体实现下降到各个子类中实现，父类仅仅指定这些方法何时被调用，从而不再关心有多少子类，实现了“对拓展开放、对修改关闭”。当然你也会发现方法的调用者不能再偷懒，传递字符串就能拿到合适的披萨类型了，现在

当然，使用工厂方法模式也有代价，对于 n 个披萨类型，现在我们需要 2n+1 个类了。其中 n 个 `Pizza` 的子类，1 个 `APizzaStore` 的抽象类负责制定流程框架，n 个 `APizzaStore` 的子类负责实现细节。

以还是那句话，工厂方法模式和 **工厂** 半毛钱关系都没有。它只是一种 OOP 下的编程技巧，在任何场景下都有可能使用。但这个技巧和语言有点关系，比如这里的例子是 Java 语言。我们知道 Java 可以把一个类标记为 `abstract`，虚拟类有一个虚拟方法。子类如果想变成具体类就必须实现这个虚拟方法。但是在 OC 中并没有虚拟类的概念，父类的空方法子类完全可以不实现，那么就无法在编译时作出这些规定，只能靠文档和 Code Review 来督促(父类方法抛错是运行时)。

## 抽象工厂模式

最后聊聊抽象工厂模式，依我愚见，抽象工厂模式和 **工厂** 更没关系。

在某些极端情况下，披萨的种类可能会特别多，但并不是毫无规律的多。可能会出现可以归类的情况。比如我们考虑两个维度，一个是披萨的产地，可以是中国、美国、印度、日本等等，另一个是披萨的口味，它的数量有限，只有麻辣、微辣和不辣三种:

![](http://images.bestswifter.com/1492518813.png)

这里画了个很简单图表，一共有 15 种披萨。我们可以用中国不辣、日本不辣、印度不辣来描述三种披萨，也可以先建立三个披萨工场，分别用来生产不辣、微辣、麻辣的披萨，然后用不辣工厂的中国披萨、日本披萨、印度披萨来描述上述三种披萨。这有点类似于数学里面提取公因数的概念。

假设我们建立了三个工厂，分别生产不辣、微辣、麻辣的披萨，以微辣工厂为例:

```java
//snippet 4
abstract class APizzaFactory{  
    public abstract ChinesePizza createChinesePizza();  
    public abstract JapanesePizza createJapanesePizza();    
    public abstract AmericanPizza createAmericanPizza();  
    // ...
} 

class HotFactory extends APizzaFactory{  
    public abstract ChinesePizza createChinesePizza() {  
        return new HotChinesePizza();  
    }  
    public abstract JapanesePizza createJapanesePizza() {  
        return new HotJapanesePizza();  
    }  
    public abstract AmericanPizza createAmericanPizza() {  
        return new HotAmericanPizza();  
    }
}  

HotFactory hotFactory = new HotFactory();  
ChinesePizza pizza = hotFactory.createChinesePizza();  
```

这样我们在创建披萨的时候就不用了解 15 个具体子类了，只要了解三种工厂和五个种类。换句话说我们把一个很长的一维数组(1 x 15)转化了二维数组，每个维度的长度都不大(3 x 5)。我们还可以换一个思路，比如建立五个工厂，分别表示不同国家，然后每个工厂可以生产三种不同口味的披萨。

那么到底是以口味为标准建立工厂，还是以国家为标准呢？我的建议是尽量减少新增工厂的可能性。比如上图中可以看到，新增口味的成本是添加五个国家披萨在这个新口味下的实现，而新增国家的成本仅仅是在已有的三个工厂中各增加一个方法。可见新增工厂(口味)比新增产品(国家)更麻烦一些。所以在上述例子中，个人建议针对不同的口味建立工厂，在实际项目中作出正确的选择应该也不会太困难。

考虑到披萨子类过多，而且大部分可以分类，在实际项目中为了节省代码量，我们还可以用反射的方式来动态获取类并生成实例。

总之，抽象工厂模式名字很玄乎，但是概念也很简单，就是用二维数组的思想来简化数量多、但可以分类的数据。

# 总结

这是学习设计模式的第一篇文章，从比较简单的工厂方法开始讲起。重点是忽略设计模式的表象，挖掘背后的原理。比如工厂模式就和工厂、创建对象没啥关系:

1. 简单工厂模式: 具体逻辑由具体类处理，减少不必要依赖，方便代码复用
2. 工厂方法模式: 父类定框架，子类做实现，利用多态的特点实现对拓展开放，对修改关闭
3. 抽象工厂模式: 借用二维数组的概念对复杂的子类做分类，简化业务逻辑。

# 参考文档

本文写作过程中参考了以下两篇文章，但他们并不权威:

1. [深入浅出工厂设计模式](https://segmentfault.com/p/1210000009074890/read)
2. [简单工厂、工厂方法、抽象工厂、策略模式、策略与工厂的区别](http://lh-kevin.iteye.com/blog/1981574)
