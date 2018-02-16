## 背景

对于单纯做前端或者后端的同学来说，一般很难接触到编码问题，因为在同一个平台上，一般都是使用同一种编码方式，自然问题不大。但对于写爬虫的同学来说，编码很可能是遇到的第一个坑。这是因为字符串无法直接通过网络被传输(也不能直接被存储)，需要先转换成二进制格式，再被还原。因此凡是涉及到通过网络传输字符的地方，通常都容易遇到编码问题。

## 概念定义

为了方便解释，我们首先来定义一些概念。每个开发者都知道字符串，它是一些字符的集合， 比如  `hello world` 就是一个最常见的字符串。相对来说，**字符** 比较难定义一些。从语义上来讲，它是组成字符串的最基本单位，比如这里的字母、空格，以及标点、特定语言(中文、日文)、emoji 符号等等。

字符是语言中的概念，但是计算机只认识 0 和 1 这两个数字。因此要想让计算机存储、处理字符串，就必须把字符串用二进制表示出来。在 ASCII 码中，每个英文字母都有自己对应的数字。我们通常把 ASCII 码称为**字符集**，也就是字符的集合。了解 ASCII 码的同学应该都知道小写字母 a 可以用 97 来表示，97 也被称为字符 `a` 在 ASCII 字符集中的**码位**。

如果要设计一种密码，最简单的方式就是把字母转换成它在 ASCII 码中的码位再发送，接受者则查找 ASCII 码表，还原字符。可见把字符转换成码位的过程类似于加密(encrypt)，我们称之为**编码**(encode)，反则则类似于解密，我们称之为**解码**(decode)

![图片](http://images.bestswifter.com/ascii-encode.png)

## 编码方式

字符转换成码位的过程是**编码**，这个过程有无数种实现方式。比如 `a -> 97`、`b -> 98` 这种就是 ASCII 编码，因为 255 = 2 ^ 8，所以所有 ASCII 编码下的码位恰好都可以由一个字节表示。

ASCII 比较诞生得比较早，随着越来越多的国家开始使用计算机，0-255 这么点码位肯定不够用了。比如中国人为了展示汉字，发明了 GB2312 编码。GB2312编码完全向下兼容 ASCII 编码，也就是说 **所有 ASCII 字符集中的字符，它在 GB2312 编码下的码位与 ASCII 编码下的码位完全一致**，而中文则由两个字节表示，这也就是为什么早期我们一般认为一个中文等于两个英文的原因。

## Unicode

除了中国人自己的编码方式，各个地区的人也都根据自己的语言拓展了相应的编码方式。那么问题就来了， 给你一个码位 `0xEE 0xDD`，它到底表示什么字符，取决于它是用哪种编码方式编码的。这就好比你拿到了密文，但没有密码表一样。因此，要想正确显示一种语言，就必须携带这个语言的编码规范，要想正确显示世界上所有的语言，看起来就比较困难了。

因此 Unicode 实际上是一种统一的字符集规范，每一个 Unicode 字符由 6 个十六进制数字表示，因此理论上可以表示出 `16 ^ 6 = 16777216` 个字符，显然是绰绰有余了。

## Unicode 编码怎么样

看起来 Unicode 就是一种很棒的编码方式。诚然，Unicode 可以表示所有的字符，但过于全面带来的缺点就是过于庞大。对于字符 `a` 来说，如果使用 ASCII 编码，可以表示为 0x61，只要一个字节就能存下，但它的 Unicode 码位是 0x000061，需要三个字节。因此采用 Unicode 编码的英文内容，会比 ASCII 编码大三倍。这大大增加了文件本地存储时占用的空间以及传输时的体积。

因此，我们有了对 Unicode 字符再次编码的编码方式，常见的有 utf-8，utf-16 等。UTF 表示 Unicode Transfer Format，因此是针对 Unicode 字符集的一系列编码方式。utf-8 是一种变长编码，也就是说不同的 Unicode 字符在 utf-8 编码下的码位长度可能不同，如下表所示:

|Unicode 编码(16进制)|utf-8 码位(二进制)|
|:---:|:---|
|000000-00007F|0xxxxxxx|
|000080-0007FF|110xxxxx 10xxxxxx|
|000800-00FFFF|1110xxxx 10xxxxxx 10xxxxxx|
|010000-1FFFFF|11110xxx10xxxxxx10xxxxxx10xxxxxx|

这个表有两点值得注意。一个是 ASCII 字符集中的所有字符，它们的 utf-8 码位依然占用一个字节，因此 utf-8 编码下的英文字符不会向 Unicode 一样增加大小。另一个则是所有中文的 utf-8 码位都占用 3 个字节，大于 GBK 编码的 2 字节。因此如果存在明确的业务需要，是可以用 GBK 编码取代 utf-8 编码的。

![图片](http://images.bestswifter.com/Unicode.png)

尽管 utf-8 非常常用，但它可变长度的特点不仅会导致某些场景下内容过大，也为索引和随机读取带来了困难。因此在很多操作系统的内存运算中，通常使用 utf-16 编码来代替。utf-16 的特点是所有码位的最小单位都是 2 字节，虽然存在冗余，但易于索引。由于码位都是两个字节，就会存在字节序的问题。因此 utf-16 编码的字符串，一开头会有几个字节的 BOM（Byte order markd）来标记字节序，比如 `0xFF 2`(FE0x55,254) 表示 Intel CPU 的小字节序，如果不加 BOM 则默认表示大字节序。需要注意的是，某些应用会给 utf-8 编码的字节也加上 BOM。

虽然看起来问题变得复杂了，为了存储/传输一个字符，竟然需要两次编码，但别忘了，Unicode 编码是通用的，因此可以内置于操作系统内部。所以我们平时所谓的对字符串进行 utf-8 编码，其实说的是对字符串的 Unicode 码位进行 utf-8 编码。

这一点在 python3 中得到了充分的体现，字符串由字符组成，每一个字符都是一个 Unicode 码位。

## 编解码错误处理

如果把编解码理解成利用密码表进行加解密，那么就容易理解，为什么编码和解码过程都是易错的。

如果被编码的 Unicode 字符，在某种编码中根本没有列出编码方式，这个字符就无法被编码:

```python
city = 'São Paulo'
b_city = city.encode('cp437')
# UnicodeEncodeError: 'charmap' codec can't encode character '\xe3' in position 1: character maps to <undefined>

b_city = city.encode('cp437', errors='ignore') 
# b'So Paulo'

b_city = city.encode('cp437', errors='replace')
# b'S?o Paulo'
```

同理，如果被解码的码位，在编码表中找不到与之对应的字符，这个码位就无法被解码:

```python
octets = b'Montr\xe9al'
s_octest1 = octets.decode('utf8')
# UnicodeDecodeError: 'utf-8' codec can't decode byte 0xe9 in position 5: invalid continuation byte

s_octest1 = octets.decode('cp1252')
# Montréal

s_octest1 = octets.decode('iso8859_7')
# Montrιal

s_octest1 = octets.decode('utf8', errors='replace')
# Montr�al
```

python 的解决方案是，`encode` 和 `decode` 函数都有一个参数 `errors` 可以控制如何处理无法被编、解码的内容。它的值可以是 `ignore`(忽略这个错误并继续执行)，也可以是 `replace`(用系统的占位符填充)。

一般来说，无法从码位推断出编码方式，就像你不可能从密文推断出加密方式一样。但是某些编码方式会留下非常显著的特征，一旦这些特征频繁出现，基本就可以断定编码方式。Python 提供了一个名为 `Chardet` 的包，可以帮助开发者推断出编码方式，并且会给出相应的置信度。置信度越高，说明是这种编码方式的可能性越大。

```python
octets = b'Montr\xe9al'
chardet.detect(octets)
# {'encoding': 'ISO-8859-1', 'confidence': 0.73, 'language': ''}

octets.decode('ISO-8859-1')
# Montréal
```

## 总结

1. 编码是为了把人类人类可读的字符转换成计算机容易存储和传输的二进制，解码反之，编码后得到的结果称之为码位。
2. Unicode 是一种通用字符集，从字符到 Unicode 字符集中码位的转换也可以叫做 Unicode 编码
3. Unicode 编码对英文字符不友好，因此出现了针对 Unicode 码位的再次编码，比如 utf-8，希望在节省空间的同时保留强大的表达能力
4. 各个编码之间的关系如下图所示:

![](http://images.bestswifter.com/encode-relationship.png)
