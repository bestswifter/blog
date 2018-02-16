这篇文章由一个简单的问题引出:

> 有两个字典，分别存有 100 条数据和 10000 条数据，如果用一个不存在的 key 去查找数据，在哪个字典中速度更快？ 

有些计算机常识的读者都会立刻回答: “一样快，底层都用了哈希表，查找的时间复杂度为 O(1)”。然而实际情况真的是这样么？

答案是否定的，存在少部分情况两者速度不一致，本文首先对哈希表做一个简短的总结，然后思考 Java 和 Redis 中对哈希表的实现，最后再得出结论，如果对某个话题已经很熟悉，可以直接跳到文章末尾的对比和总结部分。

## 哈希表概述

Objective-C 中的字典 `NSDictionary` 底层其实是一个哈希表，实际上绝大多数语言中字典都通过哈希表实现，比如我曾经分析过的 [Swift 字典的实现原理](http://www.jianshu.com/p/0caa1076b751)。

在讨论哈希表之前，先规范几个接下来会用到的概念。哈希表的本质是一个数组，数组中每一个元素称为一个箱子(bin)，箱子中存放的是键值对。

哈希表的存储过程如下:

1. 根据 key 计算出它的哈希值 h。
2. 假设箱子的个数为 n，那么这个键值对应该放在第 **(h % n)** 个箱子中。
3. 如果该箱子中已经有了键值对，就使用开放寻址法或者拉链法解决冲突。

在使用拉链法解决哈希冲突时，每个箱子其实是一个链表，属于同一个箱子的所有键值对都会排列在链表中。

哈希表还有一个重要的属性: 负载因子(load factor)，它用来衡量哈希表的 **空/满** 程度，一定程度上也可以体现查询的效率，计算公式为:

> 负载因子 = 总键值对数 / 箱子个数

负载因子越大，意味着哈希表越满，越容易导致冲突，性能也就越低。因此，一般来说，当负载因子大于某个常数(可能是 1，或者 0.75 等)时，哈希表将自动扩容。

哈希表在自动扩容时，一般会创建两倍于原来个数的箱子，因此即使 key 的哈希值不变，对箱子个数取余的结果也会发生改变，因此所有键值对的存放位置都有可能发生改变，这个过程也称为重哈希(rehash)。

哈希表的扩容并不总是能够有效解决负载因子过大的问题。假设所有 key 的哈希值都一样，那么即使扩容以后他们的位置也不会变化。虽然负载因子会降低，但实际存储在每个箱子中的链表长度并不发生改变，因此也就不能提高哈希表的查询性能。

基于以上总结，细心的读者可能会发现哈希表的两个问题:

1. 如果哈希表中本来箱子就比较多，扩容时需要重新哈希并移动数据，性能影响较大。
2. 如果哈希函数设计不合理，哈希表在极端情况下会变成线性表，性能极低。

我们分别通过 Java 和 Redis 的源码来理解以上问题，并看看他们的解决方案。

## Java 8 中的哈希表

JDK 的代码是开源的，可以从[这里](http://download.java.net/openjdk/jdk8/)下载到，我们要找的 HashMap.java 文件的目录在 `openjdk/jdk/src/share/classes/java/util/HashMap.java`。

HashMap 是基于 HashTable 的一种数据结构，在普通哈希表的基础上，它支持多线程操作以及空的 key 和 value。

在 HashMap 中定义了几个常量:

```java
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
static final int MAXIMUM_CAPACITY = 1 << 30;
static final float DEFAULT_LOAD_FACTOR = 0.75f;
static final int TREEIFY_THRESHOLD = 8;
static final int UNTREEIFY_THRESHOLD = 6;
static final int MIN_TREEIFY_CAPACITY = 64;
```

依次解释以上常量:

1. `DEFAULT_INITIAL_CAPACITY`: 初始容量，也就是默认会创建 16 个箱子，箱子的个数不能太多或太少。如果太少，很容易触发扩容，如果太多，遍历哈希表会比较慢。
2. `MAXIMUM_CAPACITY`: 哈希表最大容量，一般情况下只要内存够用，哈希表不会出现问题。
3. `DEFAULT_LOAD_FACTOR`: 默认的负载因子。因此初始情况下，当键值对的数量大于 `16 * 0.75 = 12` 时，就会触发扩容。
4. `TREEIFY_THRESHOLD`: 上文说过，如果哈希函数不合理，即使扩容也无法减少箱子中链表的长度，因此 Java 的处理方案是当链表太长时，转换成红黑树。这个值表示当某个箱子中，链表长度大于 8 时，有可能会转化成树。
5. `UNTREEIFY_THRESHOLD`: 在哈希表扩容时，如果发现链表长度小于 6，则会由树重新退化为链表。
6. `MIN_TREEIFY_CAPACITY`: 在转变成树之前，还会有一次判断，只有键值对数量大于 64 才会发生转换。这是为了避免在哈希表建立初期，多个键值对恰好被放入了同一个链表中而导致不必要的转化。

学过概率论的读者也许知道，理想状态下哈希表的每个箱子中，元素的数量遵守泊松分布:

![](https://o8ouygf5v.qnssl.com/1470319630.png )

当负载因子为 0.75 时，上述公式中 λ 约等于 0.5，因此箱子中元素个数和概率的关系如下:

| 数量 | 概率 |
| :--: |:-----:|
| 0 | 0.60653066 |
| 1 | 0.30326533 |
| 2 | 0.07581633 |
| 3 | 0.01263606 |
| 4 | 0.00157952 |
| 5 | 0.00015795 |
| 6 | 0.00001316 |
| 7 | 0.00000094 |
| 8 | 0.00000006 |

这就是为什么箱子中链表长度超过 8 以后要变成红黑树，因为在正常情况下出现这种现象的几率小到忽略不计。一旦出现，几乎可以认为是哈希函数设计有问题导致的。

Java 对哈希表的设计一定程度上避免了不恰当的哈希函数导致的性能问题，每一个箱子中的链表可以与红黑树切换。

## Redis

Redis 是一个高效的 key-value 缓存系统，也可以理解为基于键值对的数据库。它对哈希表的设计有非常多值得学习的地方，在不影响源代码逻辑的前提下我会尽可能简化，突出重点。

### 数据结构

在 Redis 中，字典是一个 `dict` 类型的结构体，定义在 `src/dict.h` 中:

```c++
typedef struct dict {
    dictht ht[2];
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
} dict;
```

这里的 `dictht` 是用于存储数据的结构体。注意到我们定义了一个长度为 2 的数组，它是为了解决扩容时速度较慢而引入的，其原理后面会详细介绍，`rehashidx` 也是在扩容时需要用到。先看一下 `dictht`  的定义:

```c++
typedef struct dictht {
    dictEntry **table;
    unsigned long size;
    unsigned long used;
} dictht;
```

可见结构体中有一个二维数组 `table`，元素类型是 `dictEntry`，对应着存储的一个键值对:

```c++
typedef struct dictEntry {
    void *key;
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next;
} dictEntry;
```

从 `next` 指针以及二维数组可以看出，Redis 的哈希表采用拉链法解决冲突。

整个字典的层次结构如下图所示:

![](http://images.bestswifter.com/1470323082.png?watermark/2/text/QGJlc3Rzd2lmdGVy/font/5a6L5L2T/fontsize/500/fill/I0ZCRkJGQw==/dissolve/100/gravity/SouthEast/dx/10/dy/10 )

### 添加元素

向字典中添加键值对的底层实现如下:

```c++
dictEntry *dictAddRaw(dict *d, void *key) {
    int index;
    dictEntry *entry;
    dictht *ht;

    if (dictIsRehashing(d)) _dictRehashStep(d);
    if ((index = _dictKeyIndex(d, key)) == -1)
        return NULL;

    ht = dictIsRehashing(d) ? &d->ht[1] : &d->ht[0];
    entry = zmalloc(sizeof(*entry));
    entry->next = ht->table[index];
    ht->table[index] = entry;
    ht->used++;

    dictSetKey(d, entry, key);
    return entry;
}
```

`dictIsRehashing` 函数用来判断哈希表是否正在重新哈希。所谓的重新哈希是指在扩容时，原来的键值对需要改变位置。为了优化重哈希的体验，Redis 每次只会移动一个箱子中的内容，下一节会做详细解释。

仔细阅读指针操作部分就会发现，新插入的键值对会放在箱子中链表的头部，而不是在尾部继续插入。这个细节上的改动至少带来两个好处:

1. 找到链表尾部的时间复杂度是 O(n)，或者需要使用额外的内存地址来保存链表尾部的位置。头插法可以节省插入耗时。
2. 对于一个数据库系统来说，最新插入的数据往往更有可能频繁的被获取。头插法可以节省查找耗时。

### 增量式扩容

所谓的增量式扩容是指，当需要重哈希时，每次只迁移一个箱子里的链表，这样扩容时不会出现性能的大幅度下降。

为了标记哈希表正处于扩容阶段，我们在 `dict` 结构体中使用 `rehashidx` 来表示当前正在迁移哪个箱子里的数据。由于在结构体中实际上有两个哈希表，如果添加新的键值对时哈希表正在扩容，我们首先从第一个哈希表中迁移一个箱子的数据到第二个哈希表中，然后键值对会被插入到第二个哈希表中。

在上面给出的 `dictAddRaw` 方法的实现中，有两句代码:

```c++
if (dictIsRehashing(d)) _dictRehashStep(d);
// ...
ht = dictIsRehashing(d) ? &d->ht[1] : &d->ht[0];
```

第二句就是用来选择插入到哪个哈希表中，第一句话则是迁移 `rehashidx` 位置上的链表。它实际上会调用 `dictRehash(d,1)`，也就是说是单步长的迁移。`dictRehash` 函数的实现如下:

```objc
int dictRehash(dict *d, int n) {
    int empty_visits = n*10; /* Max number of empty buckets to visit. */

    while(n-- && d->ht[0].used != 0) {
        dictEntry *de, *nextde;

        while(d->ht[0].table[d->rehashidx] == NULL) {
            d->rehashidx++;
            if (--empty_visits == 0) return 1;
        }
        de = d->ht[0].table[d->rehashidx];
        /* Move all the keys in this bucket from the old to the new hash HT */
        while(de) {
            unsigned int h;

            nextde = de->next;
            /* Get the index in the new hash table */
            h = dictHashKey(d, de->key) & d->ht[1].sizemask;
            de->next = d->ht[1].table[h];
            d->ht[1].table[h] = de;
            d->ht[0].used--;
            d->ht[1].used++;
            de = nextde;
        }
        d->ht[0].table[d->rehashidx] = NULL;
        d->rehashidx++;
    }

    /* Check if we already rehashed the whole table... */
    if (d->ht[0].used == 0) {
        zfree(d->ht[0].table);
        d->ht[0] = d->ht[1];
        _dictReset(&d->ht[1]);
        d->rehashidx = -1;
        return 0;
    }

    return 1;
}
```

这段代码比较长，但是并不难理解。它由一个 while 循环和 if 语句组成。在单步迁移的情况下，最外层的 while 循环没有意义，而它内部又可以分为两个 while 循环。

第一个循环用来更新 `rehashidx` 的值，因为有些箱子为空，所以 `rehashidx` 并非每次都比原来前进一个位置，而是有可能前进几个位置，但最多不超过 10。第二个循环则用来复制链表数据。

最外面的 if 判断中，如果发现旧哈希表已经全部完成迁移，就会释放旧哈希表的内存，同时把新的哈希表赋值给旧的哈希表，最后把 `rehashidx` 重新设置为 -1，表示重哈希过程结束。

### 默认哈希函数

与 Java 不同的是，Redis 提供了 `void *` 类型 key 的哈希函数，也就是通过任何类型的 key 的指针都可以求出哈希值。具体算法定义在 `dictGenHashFunction` 函数中，由于代码过长，而且都是一些位运算，就不展示了。

它的实现原理是根据指针地址和这一块内存的长度，获取内存中的值，并且放入到一个数组当中，可见这个数组仅由 0 和 1 构成。然后再对这些数字做哈希运算。因此即使两个指针指向的地址不同，但只要其中内容相同，就可以得到相同的哈希值。

## 归纳对比

首先我们回顾一下 Java 和 Redis 的解决方案。

Java 的长处在于当哈希函数不合理导致链表过长时，会使用红黑树来保证插入和查找的效率。缺点是当哈希表比较大时，如果扩容会导致瞬时效率降低。

Redis 通过增量式扩容解决了这个缺点，同时拉链法的实现(放在链表头部)值得我们学习。Redis 还提供了一个经过严格测试，表现良好的默认哈希函数，避免了链表过长的问题。

Objective-C 的实现和 Java 比较类似，当我们需要重写 `isEqual()` 方法时，还需要重写 `hash` 方法。这两种语言并没有提供一个通用的、默认的哈希函数，主要是考虑到 `isEqual()` 方法可能会被重写，两个内存数据不同的对象可能在语义上被认为是相同的。如果使用默认的哈希函数就会得到不同的哈希值，这两个对象就会同时被添加到 `NSSet` 集合中，这可能违背我们的期望结果。

根据我的了解，Redis 并不支持重写哈希方法，难道 Redis 就没有考虑到这个问题么？实际上还要从 Redis 的定位说起。由于它是一个高效的，Key-Value 存储系统，它的 key 并不会是一个对象，而是一个用来唯一确定对象的标记。

一般情况下，如果要存储某个用户的信息，key 的值可能是这样: `user:100001`。Redis 只关心 key 的内存中的数据，因此只要是可以用二进制表示的内容都可以作为 key，比如一张图片。Redis 支持的数据结构包括哈希表和集合(Set)，但是其中的数据类型只能是字符串。因此 Redis 并不存在对象等同性的考虑，也就可以提供默认的哈希函数了。

Redis、Java、Objective-C 之间的异同再次证明了一点:

> 没有完美的架构，只有满足需求的架构。

## 总结

回到文章开头的问题中来，有两个字典，分别存有 100 条数据和 10000 条数据，如果用一个不存在的 key 去查找数据，在哪个字典中速度更快？ 

完整的答案是:

在 Redis 中，得益于自动扩容和默认哈希函数，两者查找速度一样快。在 Java 和 Objective-C 中，如果哈希函数不合理，返回值过于集中，会导致大字典更慢。Java 由于存在链表和红黑树互换机制，搜索时间呈对数级增长，而非线性增长。在理想的哈希函数下，无论字典多大，搜索速度都是一样快。

最后，整理了一下本文提到的知识点，希望大家读完文章后对以下问题有比较清楚透彻的理解:

1. 哈希表中负载因子的概念
2. 哈希表扩容的过程，以及对查找性能的影响
3. 哈希表扩容速度的优化，拉链法插入新元素的优化，链表过长时的优化
4. 不同语言、使用场景下的取舍

## 参考资料
1. [OpenJDK Source Release](http://download.java.net/openjdk/jdk8/)
2. [HashMap vs Hashtable vs HashSet](http://www.pakzilla.com/2009/08/24/hashmap-vs-hashtable-vs-hashset/)
3. [泊松分布](http://baike.baidu.com/link?url=NQWCkIcXatyKrz5bfKLQLOWMBvyGAK7JXudS6_Av5Y3gGe08pvWAE_Eg2J2Q0i5MJUxVwIVtR__0EydMo-xJXa)
4. [Redis Source code](https://github.com/antirez/redis)
5. [An introduction to Redis data types and abstractions](http://redis.io/topics/data-types-intro)