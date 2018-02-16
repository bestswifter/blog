绝大多数 iOS 开发者在学习 runtime 时都阅读过 runtime.h 文件中的这段代码:

```objc
struct objc_class {
    Class isa  OBJC_ISA_AVAILABILITY;

#if !__OBJC2__
    Class super_class                                        OBJC2_UNAVAILABLE;
    const char *name                                         OBJC2_UNAVAILABLE;
    long version                                             OBJC2_UNAVAILABLE;
    long info                                                OBJC2_UNAVAILABLE;
    long instance_size                                       OBJC2_UNAVAILABLE;
    struct objc_ivar_list *ivars                             OBJC2_UNAVAILABLE;
    struct objc_method_list **methodLists                    OBJC2_UNAVAILABLE;
    struct objc_cache *cache                                 OBJC2_UNAVAILABLE;
    struct objc_protocol_list *protocols                     OBJC2_UNAVAILABLE;
#endif

} OBJC2_UNAVAILABLE;
```

可以看到其中保存了类的实例变量，方法列表等信息。

不知道有多少读者思考过 `OBJC2_UNAVAILABLE` 意味着什么。其实早在 2006 年，苹果在 WWDC 大会上就发布了 [Objective-C 2.0](https://en.wikipedia.org/wiki/Objective-C#Objective-C_2.0)，其中的改动包括 Max OS X 平台上的垃圾回收机制(现已废弃)，runtime 性能优化等。

这意味着上述代码，以及任何带有 `OBJC2_UNAVAILABLE` 标记的内容，都已经在 2006 年就永远的告别了我们，只停留在历史的文档中。

## Category 的原理

虽然上述代码已经过时，但仍具备一定的参考意义，比如 `methodLists` 作为一个二级指针，其中每个元素都是一个数组，数组中的每个元素则是一个方法。

接下来就介绍一下 category 的工作原理，在美团的技术博客 [深入理解Objective-C：Category](http://tech.meituan.com/DiveIntoCategory.html) 中已经有了非常详细的解释，然而可能由于时间问题，其中的不少内容已经过时，我根据目前最新的版本(objc-680) 做一些简单的分析，为了便于阅读，在不影响代码逻辑的前提下有可能删除部分无关紧要的内容。

### 概述

首先 runtime 依赖于 dyld 动态加载，在 objc-os.mm 文件中可以找到入口，它的调用栈简单整理如下:

```objc
void _objc_init(void)
└──const char *map_2_images(...)
    └──const char *map_images_nolock(...)
        └──void _read_images(header_info **hList, uint32_t hCount)
```

以上四个方法可以理解为 runtime 的初始化过程，我们暂且不深究。在 `_read_images` 方法中有如下代码:

```objc
if (cat->classMethods  ||  cat->protocols  
    /* ||  cat->classProperties */) {
    addUnattachedCategoryForClass(cat, cls->ISA(), hi);
    if (cls->ISA()->isRealized()) {
        remethodizeClass(cls->ISA());
    }
}
```

根据注释可见苹果曾经计划利用 category 来添加属性。在 `addUnattachedCategoryForClass` 方法中会找到当前类的所有 category，然后在 `remethodizeClass` 真正的去做处理。不过到目前为止还没有接触到相关的 category 处理，我们继续沿着调用栈向下走:

```objc
void _read_images(header_info **hList, uint32_t hCount)
└──static void remethodizeClass(Class cls)
    └──static void attachCategories(Class cls, category_list *cats, bool flush_caches)
```

这里的 `attachCategories` 就是处理 category 的核心所在，不过在阅读这段代码之前，我们有必要先熟悉一下相关的数据结构。

### Category 相关的数据结构

首先来了解一下一个 Category 是如何存储的，在 objc-runtime-new.h 中可以看到如下定义，我只列出了其中成员变量:

```objc
struct category_t {
    const char *name;
    classref_t cls;
    struct method_list_t *instanceMethods;
    struct method_list_t *classMethods;
    struct protocol_list_t *protocols;
    struct property_list_t *instanceProperties;
};
```

可见一个 category 持有了一个 `method_list_t` 类型的数组，`method_list_t` 又继承自 `entsize_list_tt`，这是一种泛型容器:

```objc
struct method_list_t : entsize_list_tt<method_t, method_list_t, 0x3> {
    // 成员变量和方法
};

template <typename Element, typename List, uint32_t FlagMask>
struct entsize_list_tt {
    uint32_t entsizeAndFlags;
    uint32_t count;
    Element first;
};
```

这里的 `entsize_list_tt` 可以理解为一个容器，拥有自己的迭代器用于遍历所有元素。 `Element` 表示元素类型，`List` 用于指定容器类型，最后一个参数为标记位。

虽然这段代码实现比较复杂，但仍可了解到 `method_list_t` 是一个存储 `method_t` 类型元素的容器。`method_t` 结构体的定义如下:

```objc
struct method_t {
    SEL name;
    const char *types;
    IMP imp;
};
```

最后，我们还有一个结构体 `category_list` 用来存储所有的 category，它的定义如下:

```objc
struct locstamped_category_list_t {
    uint32_t count;
    locstamped_category_t list[0];
};
struct locstamped_category_t {
    category_t *cat;
    struct header_info *hi;
};
typedef locstamped_category_list_t category_list;
```

除了标记存储的 category 的数量外，`locstamped_category_list_t` 结构体还声明了一个长度为零的数组，这其实是 C99 中的一种写法，允许我们在运行期动态的申请内存。

以上就是相关的数据结构，只要了解到这个程度就可以继续读源码了。

### 处理 Category

对 Category 中方法的解析并不复杂，首先来看一下 `attachCategories` 的简化版代码:

```objc
static void attachCategories(Class cls, category_list *cats, bool flush_caches) {
    if (!cats) return;
    bool isMeta = cls->isMetaClass();

    method_list_t **mlists = (method_list_t **)malloc(cats->count * sizeof(*mlists));
    // Count backwards through cats to get newest categories first
    int mcount = 0;
    int i = cats->count;
    while (i--) {
        auto& entry = cats->list[i];

        method_list_t *mlist = entry.cat->methodsForMeta(isMeta);
        if (mlist) {
            mlists[mcount++] = mlist;
        }
    }

    auto rw = cls->data();

    prepareMethodLists(cls, mlists, mcount, NO, fromBundle);
    rw->methods.attachLists(mlists, mcount);
    free(mlists);
    if (flush_caches  &&  mcount > 0) flushCaches(cls);
}
```

首先，通过 while 循环，我们遍历所有的 category，也就是参数 `cats` 中的 `list` 属性。对于每一个 category，得到它的方法列表 `mlist` 并存入 `mlists` 中。

换句话说，我们将所有 category 中的方法拼接到了一个大的二维数组中，数组的每一个元素都是装有一个 category 所有方法的容器。这句话比较绕，但你可以把 `mlists` 理解为文章开头所说，旧版本的 `objc_method_list **methodLists`。

在 while 循环外，我们得到了拼接成的方法，此时需要与类原来的方法合并:

```objc
auto rw = cls->data();
rw->methods.attachLists(mlists, mcount);
```

这两行代码读不懂是必然的，因为在 Objective-C 2.0 时代，对象的内存布局已经发生了一些变化。我们需要先了解对象的布局模型才能理解这段代码。

## Objective-C 2.0 对象布局模型

### objc_class

相信读到这里的大部分读者都学习过文章开头所说的对象布局模型，因此在这一部分，我们采用类比的方法，来看看 Objective-C 2.0 下发生了哪些改变。

首先，`Class` 和 `id` 指针的定义并没有发生改变，他们一个指向类对应的结构体，一个指向对象对应的结构体:

```objc
// objc.h
typedef struct objc_class *Class;
typedef struct objc_object *id;
```

比较有意思的一点是，`objc_class` 结构体是继承自 `objc_object` 的:

```objc
struct objc_object {
    Class isa  OBJC_ISA_AVAILABILITY;
};

struct objc_class : objc_object {
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags

    class_rw_t *data() { 
        return bits.data();
    }
};
```

这一点也很容易理解，早在 Objective-C 1.0 时代，我们就知道一个对象的结构体只有 `isa` 指针，指向它所属的类。而类的结构体也有 `isa` 指针指向它的元类。因此让类结构体继承自对象结构体就很容易理解了。 

可见 Objective-C 1.0 的布局模型中，`cache` 和 `super_class` 被原封不动的移过来了，而剩下的属性则似乎消失不见。取而代之的是一个 `bits` 属性，以及 `data()` 方法，这个方法调用的其实是 `bits` 属性的 `data()` 方法，并返回了一个 `class_rw_t` 类型的结构体指针。 

### class_data_bits_t

以下是简化版 `class_data_bits_t` 结构体的定义:

```objc
struct class_data_bits_t {
    uintptr_t bits;
public:
    class_rw_t* data() {
        return (class_rw_t *)(bits & FAST_DATA_MASK);
    }
}
```

可见这个结构体只有一个 64 位的 `bits` 成员，存储了一个指向 `class_rw_t` 结构体的指针和三个标志位。它实际上由三部分组成。首先由于 Mac OS X 只使用 47 位内存地址，所以前 17 位空余出来，提供给 `retain/release 和` `alloc/dealloc` 方法使用，做一些优化。其次，由于内存对齐，指针地址的后三位都是 0，因此可以用来做标志位:

```objc
// class is a Swift class
#define FAST_IS_SWIFT           (1UL<<0)
// class or superclass has default retain/release/autorelease/retainCount/
//   _tryRetain/_isDeallocating/retainWeakReference/allowsWeakReference
#define FAST_HAS_DEFAULT_RR     (1UL<<1)
// class's instances requires raw isa
#define FAST_REQUIRES_RAW_ISA   (1UL<<2)
// data pointer
#define FAST_DATA_MASK          0x00007ffffffffff8UL
```

如果计算一下就会发现，`FAST_DATA_MASK` 这个 16 进制常量的二进制表示恰好后三位为0，且长度为47位: `11111111111111111111111111111111111111111111000`，我们通过这个掩码做按位与运算即可取出正确的指针地址。

引用 Draveness 在 [深入解析 ObjC 中方法的结构](https://github.com/Draveness/iOS-Source-Code-Analyze/blob/master/objc/%E6%B7%B1%E5%85%A5%E8%A7%A3%E6%9E%90%20ObjC%20%E4%B8%AD%E6%96%B9%E6%B3%95%E7%9A%84%E7%BB%93%E6%9E%84.md) 中的图片做一个总结:

![bits 示意图](http://images.bestswifter.com/1470142965.png )

### class_rw_t

`bits` 中包含了一个指向 `class_rw_t` 结构体的指针，它的定义如下:

```objc
struct class_rw_t {
    uint32_t flags;
    uint32_t version;

    const class_ro_t *ro;

    method_array_t methods;
    property_array_t properties;
    protocol_array_t protocols;
}
```

注意到有一个名字很类似的结构体 `class_ro_t`，这里的 'rw' 和 ro' 分别表示 'readwrite' 和 'readonly'。因为  `class_ro_t` 存储了一些由编译器生成的常量。

> These are emitted by the compiler and are part of the ABI. 

正是由于 `class_ro_t` 中的两个属性 `instanceStart` 和 `instanceSize` 的存在，保证了 Objective-C2.0 的 ABI 稳定性。因为即使父类增加方法，子类也可以在运行时重新计算 ivar 的偏移量，从而避免重新编译。

关于 ABI 稳定性的问题，本文不做赘述，读者可以参考 [Non Fragile ivars](http://www.jianshu.com/p/3b219ab86b09)。

如果阅读 `class_ro_t` 结构体的定义就会发现，旧版本实现中类结构体中的大部分成员变量现在都定义在 `class_ro_t` 和 `class_rw_t` 这两个结构体中了。感兴趣的读者可以自行对比，本文不再赘述。

`class_rw_t` 结构体中还有一个 `methods` 成员变量，它的类型是 `method_array_t`，继承自 `list_array_tt`。

`list_array_tt` 是一个泛型结构体，用于存储一些元数据，而它实际上是元数据的二维数组:

```objc
template <typename Element, typename List>{
    struct array_t {
        uint32_t count;
        List* lists[0];
    };
}
class method_array_t : public list_array_tt<method_t, method_list_t> 
```

其中 `Element` 表示元数据的类型，比如 `method_t`，而 `List` 则表示用于存储元数据的一维数组，比如 `method_list_t`。

`list_array_tt` 有三种状态:

1. 自身为空，可以类比为 `[[]]`
2. 它只有一个指针，指向一个元数据的集合，可以类比为 `[[1, 2]]`
3. 它有多个指针，指向多个元数据的集合，可以类比为 `[[1, 2], [3, 4]]`

当一个类刚创建时，它可能处于状态 1 或 2，但如果使用 `class_addMethod` 或者 category 来添加方法，就会进入状态 3，而且一旦进入状态 3 就再也不可能回到其他状态，即使新增的方法后来又被移除掉。

## 方法合并

掌握了这些 runtime 的基础只是以后就可以继续钻研剩下的 category 的代码了:

```objc
auto rw = cls->data();
rw->methods.attachLists(mlists, mcount);
```

这是刚刚卡住的地方，现在来看，`rw` 是一个 `class_rw_t` 类型的结构体指针。根据 runtime 中的数据结构，它有一个 `methods` 结构体成员，并从父类继承了 `attachLists` 方法，用来合并 category 中的方法:

```objc
void attachLists(List* const * addedLists, uint32_t addedCount) {
    if (addedCount == 0) return;
    uint32_t oldCount = array()->count;
    uint32_t newCount = oldCount + addedCount;
    setArray((array_t *)realloc(array(), array_t::byteSize(newCount)));
    array()->count = newCount;
    memmove(array()->lists + addedCount, array()->lists, oldCount * sizeof(array()->lists[0]));
    memcpy(array()->lists, addedLists, addedCount * sizeof(array()->lists[0]));
}
```

这段代码很简单，其实就是先调用 `realloc()` 函数将原来的空间拓展，然后把原来的数组复制到后面，最后再把新数组复制到前面。

在实际代码中，比上面略复杂一些。因为为了提高性能，苹果做了一些优化，比如当 List 处于第二种状态(只有一个指针，指向一个元数据的集合)时，其实并不需要在原地扩容空间，而是只要重新申请一块内存，并将最后一个位置留给原来的集合即可。

这样只多花费了很少的内存空间，也就是原来二维数组占用的内存空间，但是 `malloc()` 的性能优势会更加明显，这其实是一个空间换时间的权衡问题。

需要注意的是，无论执行哪种逻辑，参数列表中的方法都会被添加到二维数组的前面。而我们简单的看一下 runtime 在查找方法时的逻辑:

```objc
static method_t *getMethodNoSuper_nolock(Class cls, SEL sel){
    for (auto mlists = cls->data()->methods.beginLists(), 
              end = cls->data()->methods.endLists(); 
         mlists != end;
         ++mlists) {
        method_t *m = search_method_list(*mlists, sel);
        if (m) return m;
    }

    return nil;
}

static method_t *search_method_list(const method_list_t *mlist, SEL sel) {
    for (auto& meth : *mlist) {
        if (meth.name == sel) return &meth;
    }
}
```

可见搜索的过程是按照从前向后的顺序进行的，一旦找到了就会停止循环。因此 category 中定义的同名方法不会替换类中原有的方法，但是对原方法的调用实际上会调用 category 中的方法。

## 总结

读完本文后，你应该对以下内容有比较深刻的理解，排名不分先后:

1. 定义在 runtime.h 中的数据结构，如果有 `OBJC2_UNAVAILABLE` 标记则表示已经废弃。
2. Objective-C 2.0 中，类结构体的结构层次: `objc_class` -> `class_data_bits_t` -> `class_rw_t` -> `method_array_t`。
3. `class_ro_t` 结构体的作用，与 `class_rw_t` 的区别，以及和 ABI 稳定性的关系。
4. category 解析过程的调用栈以及基本的流程。
5. `method_array_t` 为什么要设计成一种类似于二维数组的数据结构，以及它的三种状态之间的关系。

## 参考资料

1. [深入理解Objective-C：Category](http://tech.meituan.com/DiveIntoCategory.html)
2. [从源代码看 ObjC 中消息的发送](https://github.com/Draveness/iOS-Source-Code-Analyze/blob/master/objc/%E4%BB%8E%E6%BA%90%E4%BB%A3%E7%A0%81%E7%9C%8B%20ObjC%20%E4%B8%AD%E6%B6%88%E6%81%AF%E7%9A%84%E5%8F%91%E9%80%81.md)
3. [深入解析 ObjC 中方法的结构](https://github.com/Draveness/iOS-Source-Code-Analyze/blob/master/objc/%E6%B7%B1%E5%85%A5%E8%A7%A3%E6%9E%90%20ObjC%20%E4%B8%AD%E6%96%B9%E6%B3%95%E7%9A%84%E7%BB%93%E6%9E%84.md)
4. [Whats is methodLists attribute of the structure objc_class for?](http://stackoverflow.com/questions/8847146/whats-is-methodlists-attribute-of-the-structure-objc-class-for)
5. [Objc与C（C++）之亲缘关系（一） Class](http://www.cnblogs.com/jiazhh/articles/3309085.html)
6. [Objective-C Runtime](http://yulingtianxia.com/blog/2014/11/05/objective-c-runtime/)