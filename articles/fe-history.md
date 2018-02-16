## 写在最开始

这是一篇面向移动端开发者的科普性文章，从前端开发的最初流程开始，结合示范代码，讨论开发流程的演变过程，希望能覆盖一部分前端开发技术栈，从而对前端开发的相关概念形成初步的认识。

本文会提供一些示范代码，然而他们无法运行，也不需要完全看懂，更多的是方便读者对相关概念和方案有更加具体形象的感受和更清晰的理解。

在写作过程中，我阅读学习了淘宝在前后端分离和前端开发技术演变方面的博客，受益匪浅，相关文章都罗列在文末的参考资料中。同时由于自身能力有限，对很多概念的理解比较浅显，甚至有误，欢迎交流指正。

## 移动端与前端的区别

在开发 App 的过程中，我们不会刻意思考开发流程，因为一切看上去都非常自然。可以本地确定的内容就直接写死，否则异步发起网络请求并动态的修改，最后把源代码编译成可执行的二进制文件供客户安装。

前端开发和移动端开发就有本质的不同了。一个网页的最终展现样式受到 HTML + CSS 的影响，而 JavaScript 脚本负责用户交互。一个页面不会被编译成可执行文件，它仅仅由几个文本文件组成，由服务端明文下发给浏览器并绘制到屏幕上。

下文中可能会反复提到“渲染”的概念，除非特别说明，它不表示解析 HTML 文档(DOM)并绘制到屏幕上这个过程，因为这一步由浏览器内核实现，普通情况下不需要做过多了解和干预。

网页可以分为静态、动态两种，前者就是一个 HTML 文件，后者可能只是一份模板，在请求时动态的计算出数据，最后拼接成 HTML 格式的字符串，这个过程就被称为渲染。

前端与移动端开发另一个显著差异是: 虽然可以在本地调试 HTML，但实际上这些 HTML 的内容需要部署在服务端，这样才能在用户发起 HTTP 请求时返回 HTML 格式的文本。

## 前端开发的混沌时代

一开始，我们没有任何工具，只能靠蛮力。我们知道 Servlet 是由 Java 编写的服务端程序，可以方便的处理 HTTP 请求和返回，因此可以直接把 HTML 文本当做字符串返回，也就是上文所说的渲染:

```java
public class HelloWorldServlet extends HttpServlet {
    @Override
    public void doGet(HttpServletRequest req, HttpServletResponse resp)
        throws ServletException, IOException {
        resp.setContentType("text/html");

        PrintWriter out = resp.getWriter();
        out.println("<html><head><title>Hello World Sample</title></head>");
        out.println("<body><h1>Hello World Title<h1><h2>" +new Date().toLocaleString() + "</h2></body></html>");
        out.flush();
    }
}
```

理论上来说，我们已经可以开始所有前端开发了，但在这混沌初开的年代，想写出一份可维护的代码恐怕得用上“洪荒之力”。把 UI 和业务逻辑写在一起是一种非常强的耦合，不利于项目日后的拓展。也无法要求每个人同时会写 Java 和前端。

## 后端 MVC

上述方案的问题之一在于逻辑混乱不清，移动开发者在入门阶段大多也经历过，解决方案比较简单:使用 MVC，把业务逻辑抽离到 Controller 中，让 View 层专注于显示 UI。

### MVC 方案实现

在前端开发领域，也有类似的技术，比如 JSP，它经过编译后变成 Servlet。在写 JSP 的时候，我们更加关心页面样式，因此代码看起来就像是 HTML，不过在 `<% %>` 代码块中可以调用 Java 函数:

```jsp
<HTML>
<HEAD>
<TITLE>JSP测试页面---HelloWorld!</TITLE>
</HEAD>
<BODY>
<%
    out.println("<h1>Hello World!</h1>");
%>
</BODY>
</HTML>
```

JSP 相当于 View 层，它从 Model 中获取数据，说的再具体一点，是使用后端的语言(比如 Java)去访问 Model 层提供的接口。

Controller 作为直接和客户端接触的模块，负责解析请求，数据校验，路由分发，结果返回等等逻辑。路由分发是指根据请求路径的不同，调用不同的 Model 和 View 提供服务。

### MVC 的缺点与改进

使用了 MVC 架构(比如大名鼎鼎的 Struts)后，似乎职责变清晰了，前端开发者负责写 JSP，后端开发者负责写 Controller 和 Model，然而在实际开发时还是有诸多问题。

首先业务逻辑依然没有严格区分，如果没有良好的编码规范，JSP 中就会混入大量业务逻辑。而一个框架存在的作用应该是让没有接受很多培训的新人也能写出合格的代码。此外，前端开发者还需要对后端逻辑有大致的了解，熟悉后端编程语言，因此也存在很多沟通、学习成本。

### 前端只写 Demo

一种解决方案是前端开发者只写 Demo，也就是提供静态的 HTML 效果给后端开发者，由后端开发者去完成视图层(比如 JSP)的开发。这样前端开发者就完全不需要了解后端知识了。

可惜这种想法很好，但是一旦付诸实现就会遇到不少问题。首先后端开发者依赖于前端的 Demo，只有看到 HTML 文件他们才可以开始实现 View 层。而前端又依赖于后端开发者完成整体的开发，才能通过网络访问来检查最终的效果，否则他们无法获取真实的数据。

更糟糕的是，一旦需求发生变动，上述流程还需要重新走一遍，前后端的交流依旧无法避免。概况来说就是前后端对接成本太高。

举个例子，在开发 App 的时候，你的同事给你发来一段代码，其中是一个本地写死的视图，然后告诉你:“这个按钮的文字要从数据库，那个图片的内容要通过网络请求获取，你把代码改造一下吧。”。于是你花了半天时间改好了代码，PM 跑来告诉你按钮文字写死就好，但是背景颜色要从数据库读取，另外，再加一个按钮吧。WTF？

显然这种开发流程效率极低，难以接受。

### HTML 模板

其实一定程度上来说，JSP 可以看做 HTML 模板的雏形。HTML 大体上肩负了两个任务: 页面框架和内容描述。所谓的 HTML 模板是指利用一种结构化的语法，表示出 HTML 的框架，同时把具体数据抽离出来。

比如 `<p>111</p>` 表示一个段落，其中内容为 “111”。如果用模板来表示，可以写作 `<p>Content</p>` 或者 `p Content` 等等。总之，不要纠结于具体语法，我们只要知道: 

> 数据 + 模板 = HTML 源码

比如在 Controller 层，可以这样调用:

```js
// 这里没有指定模板的名称，是因为使用了依赖倒置的思想，在配置文件中绑定了 Controller 对应的 View
return res.view({title: "111"});
```

模板中的代码如下: 

```js
<h1><%= Content%></h1>
```

熟悉前端开发的读者可能会发现，这其实采用了 [Sails.js](http://sailsjs.org/) + [EJS](http://www.embeddedjs.com/) 开发。前者是基于 JavaScript 的服务端应用程序，后者是基于 HTML 语法的模板，另一种风格的模板是 [Jade](http://jade-lang.com/)，不过本文的目的并不是重点介绍这些工具如何使用，就不再赘述了。

回到模板的概念上来，它相对于 JSP 的优势在于，**利用工具强行禁止前端开发者在视图层写业务逻辑**。前端开发者只需要关心 UI 实现并确定 HTML 中的变量。而后端开发者只要传入参数即可获取 HTML 格式的字符串。

模板开发的另一个好处是前后端可以同步开发。双方约定一个数据格式，前端就可以模拟出假数据并用来自测，后端也可以用生成的数据与假数据对比进行测试。同时，这个约定的数据格式扮演了契约和文档的作用，规范了双方的数据交流形式，从而节省交流的时间成本。关于更多 Mock Server 的话题，请参考 [这个连接](https://www.zhihu.com/question/35436669)。

### 后端 MVC 架构总结

使用后端 MVC 架构加上模板开发是当前比较主流的一种开发模型，但它也不是完美的。由于模板由前端开发者完成，所以要求前端开发者对后端环境(注意这里不是实现细节)有所了解。

举个简单例子，大型应用的后端要分很多文件夹，这就要求前端对代码组织结构有所了解，上传文件时需要掌握 ssh、vim 并且依赖于服务端环境。

总的来说，采用服务端 MVC 方案时，HTML 在后端渲染，整体开发流程也全部基于后端环境。因此前端工程师不可避免的需要依赖于后端(虽然使用模板后情况已经大大改善)。

## AJAX 与前端 MVC

AJAX 的全称是 Asynchronous Javascript And XML，即 “异步 JavaScript 和 XML”。它并非一个框架，而是一种编程思想。它利用 JavaScript 异步发起请求，结果以 XML 格式返回，随后 JavaScript 可以根据返回结果局部操作 DOM。

AJAX 最大的优点是不用重新加载全部数据，而是只要获取改动的内容即可。这在移动端编程中看上去是天经地义的，而前端开发则需要专门使用 AJAX 来实现，默认情况下网页的任何一处微小改动都需要重新加载整个网页。

类比移动应用就会发现，AJAX 适合做单页面更新，但是不擅长页面跳转，就像你的 app 页面跳转都是新建一个 UIViewController/Activity 而不是直接把当前页面的内容全部换掉。

得益于 AJAX 的提出，HTML 在前端渲染变成了可能。我们可以下载一个空壳 HTML 文件和一个 JavaScript 脚本，然后在 JavaScript 脚本中获取数据，为 DOM 添加节点。

这时候就出现了很多前端的 MVC 框架，比如 Backbone.js，AngularJS(姑且认为MVVM 是 MVC 的变种) 等一堆名词，你可以从 [这里](http://todomvc.com/) 找到他们各自的 Demo。以我相对熟悉的 React 为例:

```html
<!DOCTYPE html>
<html>

<head>
  <script src="../build/react.js"></script>
  <script src="../build/react-dom.js"></script>
  <script src="../build/browser.min.js"></script>
</head>

<body>
  <div id="example"></div>
  <script type="text/babel">
    var LikeButton = React.createClass({ 
      getInitialState: function() { 
        return {liked: false}; 
      }, 
      handleClick: function(event) { 
        this.setState({liked: !this.state.liked}); 
      }, 
      render: function() { 
        var text = this.state.liked ? 'like' : 'haven\'t liked'; 
        return (
          <p onClick={this.handleClick}>
            You {text} this. Click to toggle.
          </p>
        ); 
      } 
    }); 
    ReactDOM.render(
      <LikeButton />, 
      document.getElementById('example') 
    );
  </script>
</body>

</html>
```

这段代码不用完全读懂，你只要意识到，引入 React.js 这个框架后，我们就可以脱离 HTML 编程了。所有的逻辑都写在 `<script>` 标签块中的 JavaScript 代码中。

我们还创建了一个 `LikeButton` 组件，它可以拥有自己的方法，管理自己的状态等等。这里举 React 的例子可能略有不妥，因为它其实只是一个 View 层的实现方案，还需要配合 Flux 或 Redux 架构。不过也足以感受一下纯 JavaScript 开发的感觉了。

这种开发模式和移动端开发非常类似，使用 JavaScript 调用 DOM 接口来绘制视图，使用 JavaScript 来实现业务逻辑，处理数据，发起网络请求等。你完全可以理解为单纯用 JavaScript 在浏览器上开发移动应用，只不过用户下载的是 JavaScript 脚本而非传统静态语言编译后的二进制文件。

使用了前端 MVC 框架后，对于**单页应用(Single Page Application)**来说，前后端各司其职，唯一的联系就变成了 API 调用，前端开发者不再依赖后端环境，HTML 和 JavaScript 都可以放在 CDN 服务器上。这就是我们常说的 **前后端分离**  的基本概念，即前端负责展现，后端负责数据输出。

## 前后端分离的缺点

然而在我看来，上述前后端分离方案远远逊色于后端 MVC + 模板的开发流程，这是因为在实际开发中，纯 SPA 的场景并不多见，即使移动端也不是只有一个视图，而是有很多页面跳转逻辑，更何况到处都是超链接的网页呢？

我们来看看在 SPA 和多页面跳转并存的情况下，采用前端 MVC 框架进行前后端分离存在哪些不足。

### 双端 MVC 不统一

前端的 MVC 主要处理单个页面内的逻辑，而后端 MVC 框架处理的是整个 Web 服务器的逻辑。借用 [jsconf 大会](http://2014.jsconf.cn/slides/herman-taobaoweb) 上 [赫门](http://weibo.com/threeday0905) 的图片来表示:

![前后端 MVC 架构示意图](http://2014.jsconf.cn/slides/herman-taobaoweb/img/client-side-mvc.jpg)

由于前后端 MVC 框架关注的重点不同，它们的地位自然也不同。前端的 MVC 框架负责页面展示，因此它只是后端 MVC 框架的 View 层(或许只是一部分 View)。

这就会导致如下问题:

1. 前端的 Model 和后端的 Model 结构上高度类似，比如前端用到一个 `name` 属性，后端也得在 JavaBean 中定义一个 `name`。有时候为了避免前端对数据做太多逻辑处理从而导致性能下降，后端可能已经做了一些预处理。这就好比我们写 App 时，拿到的 feed 流可能是筛选、排序过的，而我们在移动端还要在 ViewModel 中做一些转化才能给 View 使用。因此，前后端逻辑上的耦合还是无法完全避免。
2. 前端的 Controller 负责页面调度，比如控件的状态管理和样式改变等。而后端的 Controller 负责调用服务，用户鉴权等。两者完全不等价。
3. 前端也有路由模块，它主要负责页面内控件之间的跳转，比如 [React-Router](https://github.com/reactjs/react-router)，而后端路由则是把不同的网络请求分发给指定的 Controller。两者逻辑也无法统一。

### SEO

SEO(搜索引擎优化)是一个移动开发者从来不考虑，但前端开发者视作生命的问题。搜索引擎的工作原理是访问每个网页，然后分析 HTML 中的标签和关键字并做记录。

一个纯异步的网页，HTML 几乎是空壳子，而偏偏关键的数据都是动态下发的，这就影响了搜索引擎爬虫的工作过程，他们会认为该网页什么都没有，即使记录下来的也是非关键数据。

早些年谷歌推出了 [Hash-bang 协议](https://developers.google.com/webmasters/ajax-crawling/docs/getting-started) 来弥补 AJAX 对 SEO 造成的负面影响，它的本质是为爬虫提供后端渲染的降级处理机制。目前谷歌的爬虫一定程度上可以阅读 JavaScript 代码并爬取相关数据，但 AJAX 在对爬虫的支持上终究不如 HTML 文本直接传输。

### 性能不够

从上文中 React 的示范代码可以看出，HTML 文件非常小，很快就被打开。但是页面的渲染逻辑直到 JavaScript 文件被下载后才能开始执行，这就会导致一段时间的白屏。

在移动网络上，前端渲染 HTML 的性能当然不如后端渲染，频繁发送 HTTP 请求也会影响加载体验，因此依赖于前端渲染的页面，在性能方面还有很大的提高空间。

### 集中 Or 分离？

很多年前，JavaScript 和 CSS 并不用单独写在外部文件中，而是直接内嵌在 HTML 代码里:

```html
<p style="background-color:green"
    onclick="javascript:myFunction()">
    This is a paragraph.
</p>
```

为了便于管理和修改，我们把 CSS 和 JavaScript 分离出来。然而到了 React 中，好像走了回头路，所有逻辑都放在 JavaScript 中。

我的理解是，这种做法适合组件化，我们很容易定义出一个组件并且重用。这种思想对于复杂的 SPA 来说或许适用，但对于并不复杂但有多个页面的网页来说，就显得太重了。引入了 React 这样的框架，以及 MVC 的结构，反而会显得过度设计，增加代码量和复杂度。

考虑到之前所说的前后端逻辑不能复用的问题，这就更容易导致性能问题。

## Node.js

### 前后端分离的哲学

至此，我们已经尝试过后端 MVC 架构，HTML 模板，前端 MVC 架构等多种方案，但结果总是难以令人满意，是时候总结原因了。

我们在进行上述实践的过程中，过度强调物理层上的前后端分离，但是忽略了两者天然就存在一定的耦合。实际上，前端开发者不仅关注 View 的实现，还应该负责一部分 Controller 中的逻辑，后端开发者则应该关心数据获取与处理，以及一些跨终端的业务逻辑。

如果页面渲染在后端实现，会导致前端开发者依赖后端实现和开发环境，后端开发者被迫熟悉前端逻辑(此时不是调用 API 而是直接生成数据并套用模板，这就要求把获取的数据转换成模板需要的数据)。

如果页面渲染全部放在前端，业务逻辑就会太靠前，从而导致不能复用。这种做法似乎有些矫枉过正了。此外，上文中也介绍了不少 AJAX 的缺点，就不赘述了。

我们似乎陷入了两难的境地，页面渲染不管是放在前端还是后端都不合适。其实这很好理解，页面渲染涉及数据逻辑和 UI，他们理应分别由前后端开发者分别负责，单独交给任何一方都显得不合适。

但如果前端工程师可以写后端代码，问题不就迎刃而解了么？实际上数据的处理可以分为两个步骤:从数据库或第三方服务获取数据，把数据转化为 View 可用的形式。前者往往和 C++/Java 服务器相关，后者则和前端模板相关，它的作用更像是 MVVM 架构中的 ViewModel。

### Node.js 分层

我在[上一篇文章中](https://bestswifter.com/nodejs/)初步介绍了 Node.js 的定位:“一个用 JavaScript 进行开发的后端应用程序框架”。因此它恰好可以完美的解决前端不了解后端逻辑和代码的问题。

Node.js 作为一个中间层，调用上游 Java 服务器提供的服务，获取数据。自身负责处理业务逻辑，路由请求，cookie 读写，页面渲染等。前端则负责应用 CSS 样式和 JavaScript 交互，类似于最早期原始的模型。

借用 [玉伯的 Web研发模式演变](https://github.com/lifesinger/blog/issues/184) 中的图片来说明:

![node.js 中间层](https://camo.githubusercontent.com/ed895cf7561cb3ec07ef74aa2dea573b57dbe219/687474703a2f2f696d672e68622e616963646e2e636f6d2f3430303931653637316230626465653236653531366163303530633663616563383038383562386131326238372d374a676646685f6677363538)

[这里](http://2014.jsconf.cn/slides/herman-taobaoweb/#/69) 还有一个解释的非常详细的表格以供参考。

### 实战应用

这不是一篇介绍 Node.js 的博客，我也不熟悉相关框架的应用，举这个例子是为了演示 Node.js 是如何做前后端分离的。

我选择了 [Sails.js 框架](http://sailsjs.org/)，项目的代码在 [Github: sails-react-example](https://github.com/mixxen/sails-react-example/blob/master/api/controllers/CommentController.js)，这里简单的分析一下。

视图都放在 `views` 目录下，采用 EJS 为模板，由 Node.js 负责在服务端渲染:

```html
<div class="container">
<h1><%= __('Comment') %>s for SAILS<small>js</small> + REACT<small>js</small></h1>
</div>
<script src="/js/commentMain.js"></script>
```

Controllers 负责页面转发与模板渲染，具体的服务转交给 Services 去完成:

```js
module.exports = {
  app : function(req, res) {
    // 如果有必要，在这里调用 Services 获取数据
    return res.view({});
  },
};
```

这个 Demo 中没有实现 Services，通常它用于和真正的后端进行交互，可以视情况选择 HTTP 或 SOAP，并对返回结果做处理。此外还有 policies/responses 模块分别对 HTTP 请求和返回内容做处理。

前端的相关代码都封装在模板层，能够与 Node.js 无缝接合。

### 风险控制

虽然我们用增加 Node.js 处理层的方式解决了前后端分离中的一些痛点，但在实际应用中还是需要考虑得更加周全。

新增一层后，势必会导致性能损耗。然而分层本就是一个在衡量得失后做出的权衡，可以通过各种优化把性能损耗降到最低。况且，在 Node.js 这一层还可以使用 BigPipe 来处理多个异步请求。

传统网页在加载页面时，首先获取 HTML，然后获取其中引用的 CSS 和 JavaScript。在服务端准备数据和网络传输过程中，浏览器需要一直等待，而 BigPipe 将页面分成若干小块，虽然每个块的加载逻辑不变，但块与块之间可以形成流水线作业，避免浏览器无意义的等待。

使用 BigPipe 技术在一定场景下可以代替 Ajax 的多个异步请求。具体介绍可以参考 [BigPipe学习研究](http://www.searchtb.com/2011/04/an-introduction-to-bigpipe.html)。

使用 Node.js 后，对前端开发者的技术要求提高了，编码工作量也会相应的增加。然而这都是工程化的必经之路，编码量增加的背后其实是沟通、维护效率的提高。

## 总结

为了处理前后端复杂的逻辑，我们尝试了使用了后端 MVC 框架来分离业务，用 HTML 模板来分离数据和 UI 样式。使用了 Ajax 技术的网页更适合单页应用，虽然做到了物理层的分离，但在处理多页面时还是有不少问题。

实际上页面渲染本就是前后端共同关心的话题，我们更应该根据业务逻辑进行前后端分离。最终选择了 Node.js，借助它使用 JavaScript 的特性，由前端工程师负责获取数据后的处理和模板的使用，分担了一部分原本逻辑上属于前端，但技术上偏向后端的任务。这种开发模式看上去像是一种倒退，其实是螺旋式的上升与返璞归真。

Node.js 基于事件和异步 I/O 的特性，以及优秀的处理高并发的能力非常适合前后端分离的场景，使用 BigPipe 等技术也足以将分层带来的损耗降到最低。选择 Node.js 做前后端分离并不一定最佳实践，但在目前看来有不错的应用，同时也需要一边探索一边前进。

## 参考资料

1. [淘宝前后端分离实践](http://2014.jsconf.cn/slides/herman-taobaoweb/#/)
2. [Web 研发模式的演变](https://github.com/lifesinger/blog/issues/184)
3. [前后端分离的思考与实践（一）](http://blog.jobbole.com/65513/)