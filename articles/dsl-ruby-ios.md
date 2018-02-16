> **阅读本文不需要预先掌握 Ruby 与 DSL 相关的知识**

# 何为 DSL

DSL(Domain Specific Language) 翻译成中文就是:“领域特定语言”。首先，从定义就可以看出，DSL 也是一种编程语言，只不过它主要是用来处理某个特定领域的问题。

广为人知的编程语言有 C、Java、PHP 等，他们被称为 GPL(General Purpose Language)，即通用目的语言。与这些语言相比，DSL 相对显得比较神秘，他们中的大多数甚至连一个名字都没有。这主要是因为 DSL 通常用来处理某个特定的、很小的领域中的问题，因此起名字这事没有太大的必要和意义。

说了这么多废话， 一定有读者在想:“能不能举个例子讲解一下，什么是 DSL”。实际上，DSL 只是对一类语言的描述，它可以非常简单:

```objc
UIView (0, 0, 100, 100) black
UILabel (50, 50, 200, 200) yellow
……
```

比如这就是我自己随便编的一个语言。它的语法看上去很奇怪，不过这不是重点。语言的根本目的是传递信息。

## 为什么要用 DSL

其实从上面的代码中已经可以比较出 DSL 和 GPL 的特点了。DSL 语法更加简洁，比如可以没有括号(这取决于你如何设计)，因此开发、阅读的效率更高。但作为代价，DSL 调试很麻烦，很难做类型检查，因此几乎难以想象可以用 DSL 开发一个大型的程序。

如果同时接触过编译型语言和脚本语言，你可以把 DSL 理解为一种比脚本语言更加轻量、灵活的语言。

## DSL 的执行过程

了解过 C 语言的开发者应该知道，从 C 语言源码到最后的可执行文件，需要经过预编译、编译(词法分析、语法分析、语义分析)、汇编、链接等步骤，最终生成 CPU 相关的机器码，也就是一堆 0 和 1。

脚本语言不需要编译(有些也可以编译)，他们在运行时被解释，当然也需要做词法分析和语法分析，最终生成机器码。

于是问题来了，自定义的 DSL 如何被执行呢？

对于词法分析和语法分析，由于语言简单，通常只是少数关键字，即使使用最简单的字符串解析，工作量和复杂度也在可接受的范围内。然而最后生成汇编代码就显得不是很有必要了，DSL 的特点不是追求执行效率，而是高效，对开发者友好。

因此一种常见的做法是，用别的语言(可以理解为宿主语言)来解析 DSL，并执行宿主语言。继续以上面的 DSL 为例，我们可以用 OC 读取这个文本文件，了解到我们要创建一个 `UIView` 对象，因此会执行以下代码:

```objc
UIView *view = [[UIView alloc] initWithFrame:CGRectMake(0, 0, 100, 100)];
view.backgroundColor = [UIColor blackColor];
```

## 如何实现 DSL

可以看到，DSL 的定义与实现毫无技术难度可言，与其说是一门语言，不如说是一种信息的标记格式，一种存储信息的协议。从这个角度来说，JSON、XML 等数据格式也可以被称为 DSL。

然而，随着关键字数量的增多，对 DSL 的解析难度迅速提高。举个简单的例子，Cocoa 框架下的控件类型有很多，因此在解析上述 DSL 时就需要考虑很多情况。这显然与 DSL 的初衷不符。

有没有一种快速实现 DSL 的方法呢？选择 ruby 一定程度上可以解决上述问题。在解释为什么偏偏选择 ruby 之前，首先介绍一些基础知识。

# Ruby

这篇文章不是用来介绍 Ruby 语法的，感兴趣的读者可以阅读 《七周七语言》 或者 《松本行弘的程序世界》这两本书的前面几个章节来入门 Ruby，进阶教程推荐 《Ruby 元编程》。

本文主要介绍为何 Ruby 经常作为宿主语言，被用来实现 DSL，用一句话概括就是:

> DSL 其实就是 Ruby 代码

上文说过，实现 DSL 的主要难度在于利用宿主语言解析 DSL 的语法，而借助 Ruby 实现的 DSL，其本身就是 Ruby 代码，只是看起来比较像 DSL。这样在执行的时候，我们完全借助了 Ruby 解释器的力量，而不需要手动分析其中的语法结构。

借用 [Creating a Ruby DSL](https://www.leighhalliday.com/creating-ruby-dsl) 这篇文章中的例子，假设我们想写一段 HTML 代码:

```html
<html>
  <body>
    <div id="container">
      <ul class="pretty">
        <li class="active">Item 1</li>
        <li>Item 2</li>
      </ul>
    </div>
  </body>
</html>
```

但又感觉手写代码太麻烦，希望简化它，所以使用一个自创的 DSL:

```ruby
html = HTMLMaker.new.document do
  body do
    div id: "container" do
      ul class: "pretty" do
        li "Item 1", class: :active
        li "Item 2"
      end
    end
  end
end
# 这个 html 变量是一个字符串，值就是上面的 HTML 文档
```

不熟悉 Ruby 语法的读者可能无法一眼看出这段看上去像是文本的内容，其实是 Ruby 代码。为什么偏偏是 Ruby，而不是 Objective-C 或者 C++ 这些语言呢？我总结为以下两点:

1. Ruby 自身的语法特性
2. Ruby 具备元编程的能力

## 语法简介

首先，Ruby 调用函数可以不用把参数放在括号中:

```ruby
def say(word)
  puts word
end

say "Hello"
# 调用 say 函数会输出 "Hello"
```

这就保证了语法的简洁，看上去像是一门 DSL。

另外要提到的一点是 Ruby 中的闭包。与 Objective-C 和 Swift 不同的是，Ruby 的闭包可以写在 `do … end` 代码块，而不是必须放在大括号中:

```ruby
(1..10).each do |i|
  puts "Number = #{i}"
end

# 输出十行，每行的格式都是 "Number = i"
```

大括号看上去就像是一门比较复杂的语言，而 `do … end` 会更容易阅读一些。

## Ruby 元编程

元编程是 Ruby 的精髓之一。我们见过很多以“元”开头的单词，比如 “元数据”、“元类”、“元信息”。这些词汇看上去很难理解，其实只要把 “元xx” 当做 “关于xx的xx”，就很容易理解了。

以元数据为例，它表示“关于数据的数据”。比如我的 ID 是 bestswifter，它是一个数据。我还可以说这个单词中有两个字母 s，一共有 11 个字母等等。这些也是数据，并且是关于数据(bestswifter)的数据，因此可以被称为元数据。

在 runtime 中经常提到的元类，也就是关于类的类。所以存储了类的方法和属性。

而所谓的元编程，自然指的就是“关于编程的编程”。编程是指用一段代码输出某个结果，而关于编程的编程则可以理解为通过编程的方式来产生这段代码。

在实际开发时，元编程通常以两种形式体现出他的威力:

1. 提供反射的功能，通过 API 来提供对运行时环境的访问和修改能力
2. 提供执行字符串格式代码的能力

在 Ruby 中，我们可以随意为任何一个类添加、修改甚至删除方法。调用不存在方法时，可以统一进行转发:

```ruby
class TestMissing
  def method_missing(m, *args, &block)
    puts "方法名：#{m}，参数：#{args}，闭包：#{block}"
  end
end

TestMissing.new.say "Hello", "World" do
  puts "Hello, world"
end
# 方法名：say，参数：["Hello", "World"]，闭包：#<Proc:0x007feeea03cb00@t.ruby:7>
```

可见，当调用不存在的方法 `say` 时，会被转发到类的 `method_missing` 方法中，并且可以很容易的获取到方法名称和参数。

有一定 iOS 开发经验的读者会立刻想到，这哪是元编程，明明就是 runtime。确实，相比于静态语言比如 Java、Swift 的反射机制而言，Objective-C 的 runtime 提供了更强大的功能，它不仅可以自省，还能动态的进行修改。当然这也是由语言特性决定的，对于静态语言来说，早在编译时期就生成了机器码，并且随后进行链接，能提供一个反射机制就很不错了，至于修改还是不要奢望。

实际上，如果我们广义的把元编程理解为:“关于编程的编程”，那么 runtime 可以理解为一种元编程的实现方式。如果狭义的把元编程理解为用代码生成代码，并且动态执行，那 runtime 就不算了。

# 利用 Ruby 实现 DSL

分别介绍了 DSL 和 Ruby 的基础概念后，我们就可以着手利用 Ruby 来实现自己的 DSL 了。

以上文生成 HTML 的 DSL 为例进行分析，为了说明问题，我把代码再次简化一下:

```ruby
html = HTMLMaker.new.document do
  body do
    div id: "container"
  end
end
```

首先我们要定义一个 `HTMLMaker` 类，并且把 `document` 方法作为入口。这个方法接收一个闭包，闭包中调用 `body` 函数，这个函数也提供了闭包，闭包中调用了 `div` 方法，并且有一个参数 `id: "container"`……

可见这其实是一个递归调用，无论是 `body` 还是 `div`，他们对应着 HTML 标签，其实都是一些并列的方法，方法可以接受若干个键值对，也就是 HTML 中标签的属性，最后再跟上一个闭包用来创建隶属于自己的子标签。

如果不用 Ruby，我们需要事先知道所有的 HTML 标签名，然后进行匹配，可想工作量有多大。而在 Ruby 中，他们都是并列关系，可以统一转发到 `method_missing` 方法中，获取方法名、参数和闭包。

我们首先解析参数，配合方法名拼凑出当前标签的字符串，然后递归调用闭包即可，核心代码如下:

```ruby
def method_missing(m, *args, &block)
    tag(m, args, &block)
end

def tag(html_tag, args, &block)
  # indent 用来记录行首的空格缩进
  # options 表示解析后的 HTML 属性，比如 id="container"， content 则是标签中的内容
  html << "\n#{indent}<#{html_tag}#{options}>#{content}"
  if block_given? # 如果传递了闭包，递归执行
    instance_eval(&block) # 递归执行闭包
    html << "\n#{indent}"
  end
  html << "</#{html_tag}>"
end
```

这里的 `instance_eval` 也是一种元编程，表示以当前实例为上下文，执行闭包，具体用途可以参考这篇文章: [Eval, module_eval, and instance_eval](https://4loc.wordpress.com/2009/05/29/eval-module_eval-and-instance_eval/)。

完整的代码意义不大，主要是细节的处理，如果感兴趣，或者没有完全理解上面这段代码的意思，可以去[原文](https://www.leighhalliday.com/creating-ruby-dsl)中查看代码，并且自行调试。

# Ruby 在 iOS 开发中的运用

Ruby 主要用来实现一些自动化脚本，而且由于 iOS 系统上没有 Ruby 解释器，所以它通常是在 Mac 系统上使用，在编译前(绝非 app 的运行时)进行一些自动化工作。

大家最熟悉的 Cocoapods 的 podfile 其实就是一份 Ruby 代码:

```ruby
target 'target_name' do
    pod 'pod_name', '~> version'
end
```

熟悉的 `do end` 代码块告诉我们，这段声明式的 pod 依赖关系，其实就是可以执行的 ruby 代码。Cocoapods 的具体实现原理可以参考 [@draveness](https://github.com/draveness) 的这篇文章: [CocoaPods 都做了什么？](http://draveness.me/cocoapods/)

在 Ruby On Rails 中有一个著名的模块: [ActiveRecord](http://guides.rubyonrails.org/active_record_basics.html)，它提供了对象关系映射的功能(ORM)。

在面向对象的语言中，我们用对象来存储数据，对象是类的实例，而在关系型数据库中，数据的抽象模型叫实体(Entity)。类和实体在一定程度上有相似性，比如都可以拥有多个属性，类对属性的增删改查操作由类对外暴露的方法实现，在关系型数据库中则是由 SQL 语句实现。

ORM 提供了对象和实体之间的对应关系，我们不再需要手写 SQL 语句，而是直接调用对象的相关方法， 这些方法的内部会生成相应的 SQL 语句并执行。可以说  ORM 框架屏蔽了数据库的具体细节， 允许我们以面向对象的方式对数据进行持久化操作。

## MetaModel

[MetaModel](https://github.com/Primix/MetaModel) 这个框架借鉴了 ActiveRecord 的功能，致力于打造一个 iOS 开发中的 ORM 框架。

在没有 ORM 时，假设有一个 `Person` 类，它有若干个属性。即使我们利用继承等面向对象的特性封装好了大量模板方法，每当增加或删除属性时，代码改动量依然不算小。考虑到实体之间还有一对一、一对多、多对多等关系，一旦关系发生变化，相关代码的变化会更大。

MetaModel 的原理就是利用 ruby 实现了一个 DSL，在 DSL 中规定了每个实体的属性和关系，这也是开发者唯一需要关心的内容。接下来的任务将完全由 MetaModel 负责，首先它会解析每个实体有哪些属性，和别的实体有哪些关系，然后生成对应的 Swift/Objective-C 代码，打包成静态库，最终以面向对象的方式向开发者暴露增删改查的 API。

在实际使用时，我们首先要写一个 Metafile 文件，它类似于 Podfile，用于规定实体的属性和关系:

```ruby
define :Article do
  attr :title
  attr :content

  has_many :comments
end

define :Comment do
  attr :content

  belongs_to :article
end
```

执行完 MetaModel 的脚本后，就会生成相关代码，并封装在静态库中，然后可以这样调用:

```swift
let article = Article.create(title: "title1", content: "content1")
article.save // 执行 INSERT 语句
article.update(title: "newTitle")
let anotherArticle = Article.find(content:"content1")
print(Article.all)
```

MetaModel 的实现原理并不复杂，但真的做起来，还是要考虑很多细节，本文不对它的内部实现做过多分析。MetaModel 已经开源，正在不断的完善中，想了解具体使用步骤或参与到 MetaModel 完善工作中的朋友，可以打开[这个页面](https://github.com/Primix/MetaModel)。