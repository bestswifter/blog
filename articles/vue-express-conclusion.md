# 两周入门 Vue + Express 的心得总结

本文主要介绍公司内部某平台从零搭建的过程中，采用的相关技术以及遇到的坑。

整个平台是完全前后端分离的，前端界面是使用 Vue 开发的单页面应用（SPA），后端则基于 Express + MySQL/Redis 开发，利用公司的云引擎来部署。

## 云服务开发模式

### 开发机时代的问题

最简单的开发流程当然是申请一台开发机，配置好 DNS 解析和 Nginx，然后配置各种开发环境，用 git 同步代并执行。

这种开发模式一两个人玩还好，但涉及到公司层面的多人开发时，很容易遇到各种问题，我随便列出了一些：

1. 因为公司内部服务非常多，可能有几百上千个，如果所有人都去改 nginx 代码肯定不现实，搞不好会把线上服务弄坏。
2. 真实开发过程中可能使用的是服务器集群，如果手动管理机器，集群的扩容和缩容都不太方便，至少现阶段没有那么多简单易使用的工具。
3. 我们需要更加自动化的服务，比如在服务器宕机、CPU/内存 等占用过高时自动通过内部交流工具或者电话、短信等方式报警，并且自动扩容。这套流程涉及到监控、报警和自动处理，如果都交给开发者来负责，必然出现大量重复的劳动。
4. 由于本机大多是使用 Mac OS 系统开发，但服务器往往是 Linux 系统。环境的不一致必然会导致问题，比如我们会发现本地明明没有问题，但是在服务端就不好使，这类问题排查起来通常还非常麻烦。
5. 业务庞大以后，会涉及到更多的流程，比如小流量实验， 线上回滚等等。这些都是真实的、可抽象封装的需求，没有必要再通过代码去控制，而是可以提供更简单的服务。

作为一个还算不上后端开发者的菜鸟，我尚且能一下子提出这些棘手的问题，可见刀耕火种的初级开发模式确实在大规模开发时是行不通的。

### PasS 简介

PaaS 表示：平台即服务（Platform as a Service），类似的名词还有 IaaS 和 Saas。可以这么简单来理解他们：

1. 我们自己买了一台服务器，这是最原始的做法，啥都不算
2. 我们去腾讯云买了一台服务器，IP 地址已经分配好了，系统也装好了，甚至还有防 DDOS 的服务，这个叫 IaaS，可以理解为买了硬件。
3. 腾讯云提供了软件开发的镜像，我们只要上传 node/golang 代码就能跑服务，这个叫 PaaS，可以理解为买了开发平台（Platform）。
4. 腾讯云提供了一套论坛代码，我们编辑一下配置和权限就能直接运营一个论坛，这个叫 SaaS，可以理解为买了一套服务（Service）。

可见，云服务其实是为用户屏蔽了硬件和系统环境的细节。以一个 Node 云服务为例，只要我们上传一段 JS 代码，就可以把服务运行起来了。那么接下来的问题就是，怎么向云服务提供代码了。

### 代码仓库管理 

最简单的方式是用 git 来同步代码，我们可以把云服务和若干个 git 仓库关联，然后每次升级服务的时候都获取到最新的代码。

但并非所有的代码拿来都能用，对于 C++/Go 语言或者使用了 webpack 等构建工具的前段项目来说，往往还需要经过编译。这个编译的过程可能不是一两行代码那么简单，我们需要单独抽出一层来处理从源码到最终代码的编译。

其实 Jenkins 就可以胜任这个工作，不过直接使用 Jenkins 会显得有些重。我们的代码仓库管理工具更多的是侧重于代码的版本管理，比如选择某个分支或者 commit 打包，以及打测试包或者正式包等等。Jenkins 的上下游任务管理等功能反而并不太需要。

不知道是业内通用的定义，还是公司自己的习惯，在前公司和现在的公司里，代码仓库管理服务都叫做 SCM 服务。

### 服务升级

说了那么多废话，抛弃了开发机这种原始的做法以后，如何上线代码呢？可以分为以下简单几步：

1. 首先本地开发完成，然后提交到 git。这一步还是必须的，同时在  SCM 上发布新的版本
2. 升级云服务，选择新版本的代码
3. 观察小流量效果，选择推全量 or 回滚代码
这种模式下开发者无需再写具体代码，只要在平台上进行一些点击操作即可，同时像监控报警、小流量实验、自动扩容等功能大大降低了开发的难度。

### 域名与端口

不管是向内部还是外部提供服务，总是需要域名。以往在使用开发机的时候，还需要设置 DNS 解析并配置 Nginx，复杂度还是比较高。

可以在公司内网直接部署好负载均衡服务器，所有的请求都会先打到这些服务器上。这就免去了 DNS 解析的配置。

至于如何解决 Nginx 配置的问题，可以选择 Consul 这个服务发现的工具。以往是我们告诉 Nginx，不同的请求该如何处理。而云服务的物理机是随时都会变的，所以它需要在启动服务的时候向 Consul 服务器注册服务，告诉服务器自己能处理哪个域名的请求。然后当接收到对应域名的请求时，Consul 服务器就知道该交给哪台物理机处理了。

这样做的好处是，域名只会和云服务绑定，大大降低了配置难度。

## Vue 概述

### 自定义端口

我使用了 Vue 的脚手架来构建应用：

```shell
npm install --global vue-cli
vue init webpack my-project
```

为了避免把本地的服务弄得乱七八糟，我选择自定义一个端口，很多人都会说，在 `package.json` 文件里面修改一下 webpack-dev-server 的启动参数就行了：

```shell
webpack-dev-server --inline --progress --port 12345 --config build/webpack.dev.conf.js
```

这个很明显就属于胡说八道了，不信可以看看命令行里的输出，是不是还显示着默认的 8080。此时虽然服务正确的运行起来了，端口也用的是自己的 12345，但命令行输出还是错的，所以需要阅读下 webpack 的逻辑，看看这个端口应该如何正确的设置。

经过一番查找，最后发现在 `config/index.js` 文件里定义了开发和生产环境中的各种配置，其中就有端口的配置，所以在这里把 8080 改成自己的端口就行了。

### ESLint 与自动格式化

因为 JS 写得少，我对 ESLint 的规范没有太多简洁，所以不会定制规则。但这并不意味着可以没有规则，在我看来 ESLint 至少有两个作用：

1. 对自己来说，写代码再也不用考虑规则了，反正都会自动格式化，大大加速了我的编码速度，尤其是从浏览器拷贝代码过来时，不管格式怎么错乱，都会自动格式化。
2. 对团队来说，有助于维护统一的代码风格，方别代码阅读和后续迭代，

网上有一坨给 Vue 项目配置 ESLint 的教程，不过基本上没有能用的。后来找到一篇 [VSCode环境下配置ESLint 对Vue单文件的检测](https://juejin.im/post/5a15402a6fb9a045167cd624) 还比较靠谱，按照这个教程做就行了。

### 打包时的资源路径

虽然开发时代码会写在各个模块里，但真正上线时肯定都会被放到一起，然后通过 uglify 等操作来做代码混淆和压缩。这些步骤已经由脚手架配置好，交给 webpack 完成了，我们只需要关注一些配置即可。

还是在 `config/index.js` 这个文件中，生产环境的配置里有一个变量叫 `assetsPublicPath`，它表示以什么样的路径去引用编译后的静态资源。注意，不管这个值怎么变，编译产物的路径是固定的：

```
dist
  |---index.html
  |---static
        |---css
        |---fonts
        |---img
        |---js
```

它的默认值是 `/`，也就在根路径中查找。此时本地打开 HTML 文件时肯定是无法正常浏览的，因为所有资源的请求都发往磁盘的根路径，当然就找不到了。

如果想在本地直接浏览，需要把 `assetsPublicPath` 的值改为 `./`，也就是在当前路径查找。但我不推荐这种做法，我们完全可以把打包命令封装一下，先执行 `npm run build` 再把它拷贝到服务器的静态资源目录中，通过后端服务来访问网页，这种模式更接近真实的线上环境，很可能就发现了一些隐藏的问题。

一般来说，对于单页面应用而言，无论这个配置是 `/` 还是 `./` 都一样，因为真正部署到服务器的时候，我们都是通过 `https://xxxxx.com` 来访问，此时的相对路径和根路径指的都是服务的根路径。

**但是！！！**上面这个结论仅仅在请求根路径的时候生效。因为 SPA 默认用的是哈希路由，也就是说，在访问某个子页面 `somepath` 时，我们请求的路径是：`https://xxxxx.com/#/somepath`。如果觉得这种写法不够美观，Vue 还提供了基于 history 模式的路由，此时访问地址变成了 `https://xxxxx.com/somepath`。

如果我们用的是相对路径访问静态资源，此时就会失败，因为 `https://xxxxx.com/path/static/css/xxx` 并不存在。所以，正确的做法是总是使用绝对路径去访问静态资源，换句话说：**不要改动 assetsPublicPath 的值，通过脚本把编译产物放到服务器的静态资源目录去访问**

### 路由

路由的配置比较简单，有两个比较有意思的概念可以介绍下。

首先是路由守卫，其实就是路由的钩子，分为全局和局部两类。比如我们可以定义一个全局的路由守卫，在跳转到任意路径前做统一的操作，也可以定义一个局部的路由守卫，只在跳转/离开某个特定路径时执行自定义操作。具体的用法参考：[Vue router 文档：导航守卫](https://router.vuejs.org/zh-cn/advanced/navigation-guards.html)

在跳转到某个页面时，除了把参数放在路径中，也可以提供一个 `params` 参数，注意此时不能再使用 `path` 来选择页面，而是必须使用路由的名称 `name`：

```js
export default new Router({
  routes: [
    {
      path: '/',
      name: 'Root',
      component: Welcome
    }
  ]
})
```

跳转到根路径的写法：

```js
this.$router.push({ 
    name: 'Root', 
    params: {
        key: 'value'
    } 
})
```

这样被跳转的页面就可以通过 `this.$route.params` 来接收参数。但这种做法并不推荐，因为这会导致组件与路由耦合，假设下次我想直接新建一个组件而不是跳转过去，就只有拓展组件的能力，让它既支持读取路由参数，又支持直接读取自己的属性 props 了。

显然这种方案不太理想，因此 Vue 的做法是：**允许组件把路由传来的参数当做自己的 props 来使用**。只要在定义路由的时候加上一行：

```js
export default new Router({
  routes: [
    {
      path: '/',
      name: 'Root',
      component: Welcome,
      props: true  // 在这里声明
    }
  ]
})
```

今后通过路由跳转到 `Welcome` 组件时，就不用去读取 `this.$route.params.property` 了，而是直接用 `this.property`。

目前基本上稍微大一点的客户端都会做组件化，这样的设计方式值得客户端学习。

### 网络层

网络层首要解决的问题自然是：**屏蔽开发环境和生产环境的后端区别**，如果只是域名的不同，还可以用相对路径来解决。但在开发模式下，本地服务肯定是跑在另一个端口，如果直接请求就会造成跨域问题，这可以通过配置 webpack 来解决。还是在 `config/index.js` 中，我们给开发模式的配置新增一个 `proxyTable` 属性，把请求交给 webpack-server 执行：

```js
dev: {
    proxyTable: {
      '/api': {
        target: 'http://0.0.0.0:18408',
        changeOrigin: true
      }
    },
}
```

这里表示将所有的 `/api` 请求都转发到本地的 18408 端口上。

由于项目中选择把所有的 api 请求都收敛到路径 `/api` 下，所以我们可以对网络库做一些全局的 hook。以 Vue 推荐的 [axios 框架](https://github.com/axios/axios)为例：

```js
import axios from 'axios'

axios.interceptors.request.use(
  function (request) {
    if (request.url.startsWith('/')) {
      request.url = '/api' + request.url
    } else if (request.url.startsWith('http://')) {
      request.url = request.url.replace('http://', 'https://')
    } else {
      // no-op
    }
    return request
  },
}
  
export default axios
```

这里我们会对左右的请求做一下区分，如果是相对路径请求，统一会加上 `/api` 的前缀。有时候我们会跨域请求别的资源，由于自己的网页是 HTTPS 的，所以需要把第三方的资源路径也转为 HTTPS，否则会被 Chrome 拦截。当然也可以在 HTML 文件中配置以下内容来解决：

```html
<meta http-equiv="Content-Security-Policy" content="upgrade-insecure-requests">
```

全局钩子能做的远远不止这些，可以根据需求灵活使用。

### Vuex 与单向数据流

Vue 中的数据不总是单向流动的，比如 `v-model` 就是一种双向绑定的写法，具体细节可以参考这篇文章：[vue中v-model和v-bind绑定数据的异同](https://www.tangshuang.net/3507.html)。

个人理解是单向和双向数据流各有千秋，双向绑定的代码写起来更爽，因为页面的改动会直接影响到 Model，而单向数据流的写法中，我们还需要自己写 `mutation`，模板代码会比较多。

但是单向数据流有一个极大的优点在于，它的所有状态变化都是可追溯的。因为只能是：**UI 事件触发 mutation，数据的改动只能通过 mutation 来完成，当数据发生变化后，实时更新 UI 界面**。

这种模式下 UI 样式其实是数据的函数，一组给定的数据只能唯一对应一种 UI，如果我们记录了所有数据的变化，就能维护一个 UI 变化的历史列表。列表中保存了每次数据发生变化时的样式快照。

这样当我们在调试时，可以找到两次 UI 不一样的快照，看看那一次执行了什么操作，导致了 UI 的变化。这给调试带了了极大的方便：

![](http://images.bestswifter.com/2018-04-26-15247490313369.jpg)

### Markdown

有些页面需要展示一些文档，最快捷的方式当然是用 markdown 来写，然后找一个插件把 markdown 转化成 HTML。这一节不会讨论转换的细节，我直接使用了一个比较知名的开源库：[miaolz123/vue-markdown](https://github.com/miaolz123/vue-markdown)：

```shell
npm install --save vue-markdown
```

注意这个空间只是做了 markdown 转 HTML，所以 CSS 文件还要自己选择，我用的是 [sindresorhus/github-markdown-css](https://github.com/sindresorhus/github-markdown-css)。

在使用中遇到几个有意思的问题。首先，编译出来的 HTML 里用的都是 `href` 这样的超链接，但如果我想做一个 wiki 页面，点击站内的文章就会跳转出去，从而破坏了 SPA 的使用体验。所以我的做法是用一个 div 包住 HTML，然后捕获点击事件，如果是页面内跳转，就转用路由：

```js
<div @click="clickHTML($event)">
    <vue-markdown :toc="true" class="markdown-body" v-bind:source="msg"></vue-markdown>
</div>

clickHTML: function (event) {
    // 页面内正常跳转，只有往外跳的时候才改用路由
    let element = event.srcElement
    if (
        element.className !== 'toc-anchor-link' &&
        element.host === window.location.host
    ) {
        this.$router.push(element.pathname)
        event.preventDefault()
    }
}
```

注意只有是同站的跳转并且不是点击了 `#` 这类的锚点，才用路由分发，别的情况都应该使用默认的策略。

在使用路由跳转时，因为命中了同一个路由，当前组件不会再经历 `create` 或者 `mounted` 这样的生命周期，所以改用 watch 的方式来获取路由参数变动的通知：

```js
watch: {
    '$route.params.file': function (id) {
        this.updateContent()
    }
},

updateContent: function () {
    this.msg = getMarkdown(this.$route.params.file || 'm-wiki')
},

function getMarkdown (filename) {
    return require('./../../../assets/markdown/' + filename + '.md')
}
```

最后用 `require`，根据路由参数动态加载 markdown 资源文件。


## Express 概述

Express 是一个比较流行的、使用中间件模式的 Node 后端框架，还是先用脚手架工具来建一个工程：

```shell
npm install express-generator -g
express --no-view project_name
```

因为项目前后端完全分离，所以这里可以用 `--no-view` 选项来建立一个只提供 api 服务的后端，如果用模板的话，可以选择 `express --view=jade test` 这种写法。

### 中间件概述

Express 的一大特点，或者说核心就是中间件。中间件的意思是指，一个请求过来以后，会**按顺序**经历我们指定的中间件， 每一个中间件可以对请求或者返回结果做任意修改，还可以选择让下一个中间件来处理，或者在自己这里结束。

这样说可能还是比较抽象，其实中间件具体到代码层面，就是一个很简单的、特定的函数，它有三个参数 `req`、`res` 和 `next`：

```js
let crossOriginMiddleWare = function (req, res, next) {
  res.header('Access-Control-Allow-Origin', '*')
  res.header('Access-Control-Allow-Headers', 'X-Requested-With')
  res.header('Access-Control-Allow-Methods', 'PUT,POST,GET,DELETE,OPTIONS')
  next()
}
```

通过 `req` 和 `res` 这两个参数，我们就可以任意处理请求和返回的内容。比如 Express 内置了一些中间件，大多是处理 `req` 的，它们会承担 url 和 json 解析、日志输出、cookie 解析等操作。这样在我们的业务逻辑中，拿到的大多是可读性比较高的数据。

使用中间件也很简单：

```js
app.use('/', crossOriginMiddleWare)
```

这里表示对于任何路径的任何一种请求方式（GET/PUT/POST/DELETE 等）都会使用中间件。

在我们的业务逻辑中，主要的操作都是针对 `res` 参数的，比如调用 `res.send('hello world')` 将数据返回给客户端。同时，作为最后一个中间件，我们一般不需要调用 `next()` 函数，表示这个请求的处理到此为止。

### 路由中间件

通过上一节对中间件的介绍，我们已经知道一个中间件其实就是处理某个对应路径的请求。假设我们要处理 `/api/1` 和 `/api/2` 这两个不同的请求，最简单的方式当然是这样写：

```js
app.use('/api/1', middleWare1)
app.use('/api/2', middleWare2)
```

但有没有可能引入命名空间的概念，简化一下代码逻辑呢：

```js
// use namespace /api
app.use('/1', middleWare1)
app.use('/2', middleWare2)
```

此时就要借助路由这个特殊的中间件了。我们可以通过 `express.Router()` 来创建路由，虽然它是一个中间件，但并不需要我们实现中间件函数，**而是可以把它当做一个局部的 app**，将其它中间件挂载到路由上：

```js
let router = express.Router()

// 这个 router 就相当于是一个局部的小型应用，从而实现了命名空间的概念
app.use('/api', router)

router.get('/1', middleWare1)
router.get('/2', middleWare2)
```

### 自动添加路由

借助路由这个特殊中间件实现了命名空间以后，我们可以为每个接口单独创建一个文件：

```js
let router = express.Router()
let middleWare1 = require('path-to-some-js-1')
let middleWare2 = require('path-to-some-js-2')
// ...
let middleWaren = require('path-to-some-js-n')

router.get('/1', middleWare1)
router.get('/2', middleWare2)
// ...
router.get('/n', middleWaren)

app.use('/api', router)
```

假设接口多了，维护起来还是挺麻烦的，其实我们可以自动读取某个文件夹下的所有 js 文件，然后导入进来。当然还要支持一下 GET 和 POST 等不同的请求方式，所以我把 api 目录设计成这样：

```
workspace
    |---api
         |---index.js
         |---get
              |---get1.js
              |---get2.js
         |---post
              |---psot1.js
              |---post2.js
```

 然后在 `index.js` 里这样写：
 
```js
var express = require('express')
var router = express.Router()

var fs = require('fs')
let path = require('path')
let basename = path.basename(__filename)
const methods = ['get', 'post']

let validFile = file => {
  return (
    file.indexOf('.') !== 0 && file !== basename && file.slice(-3) === '.js'
  )
}

let loadFile = (file, method) => {
  file = file.split('.')[0]
  var path = '/' + file // Eg: /user
  var module = './' + method + '/' + file // Eg: ./user
  router[method](path, require(module))
}

methods.map(method => {
  fs
    .readdirSync(__dirname + '/' + method)
    .filter(validFile)
    .map(file => {
      loadFile(file, method)
    })
})

module.exports = router
```

首先定义了一个 `validFile` 函数，用来判断读取的文件不是自己（index.js 肯定不能加载自己）并且文件名是合法的（比如以 .js 结尾）。

接下来定义 `loadFile` 函数，它有两个参数分别是文件名和请求方法，比如文件名是 `user.js`，方法是 get，就会拼接出路由路径为 `/user`，模块名为 `./get/user.js`，最后调用 `router.get('/user', require('./get/user.js'))` 来挂载文件。

有了这俩辅助方法以后，我们就可以遍历 `methods` 这个数组，对于每一种方法都去对应的文件夹内把文件遍历一遍，然后先过滤掉不合法文件，再加载到路由中。

这样做的好处在于，各个文件将会在运行时被自动加载进来，我们的 `index.js` 文件是保持固定的。加入我想新增一个 `/api/user` 的 GET 方法处理函数，只要在 `/api/get` 目录内新建一个 `user.js` 文件就行了。

### ORM

这个项目只是很简单的用到了 MySQL 来做数据开发，但 ORM 框架还是要使用一下的，它可以把我们从枯燥的 SQL 语句中解放出来，只要关心对象级别的增删改查。

我选择了 [Sequelize](http://docs.sequelizejs.com/) 这个库，它要求我们先定义 Model，也就是对象的各个字段。但强烈建议将这一步做成自动化的，比如使用 [sequelize-auto](https://github.com/sequelize/sequelize-auto) 这个工具，它可以读取数据库的信息，自动生成 Model。

Sequelize-auto 的使用并不算特别简单，所以我封装了一个脚本 `update_model.sh` 来做自动化，脚本主要需要处理以下几个问题：

1. sequelize-auto 依赖于 mysql 这个 npm 模块，所以要先检查下是否安装，如果没有安装需要通过这个脚本来安装
2. 执行 sequelize-auto 的主程序，指定数据库地址、端口、用户名、密码、数据库名、Model 生成路径等各种配置信息
3. 默认 sequelize-auto 会下载数据中所有的表并生成 Model 文件。如果无用的表不多，而且不常变化，可以考虑用 -T 参数来排除它们，如果用的表不多并且不常变化，那么更倾向于用 -t 参数来指定保留哪些表。但这两个参数都不支持正则匹配，所以如果上述模式都不满足，就只能自定义表名，然后下载所有 Model 再删除了

这里同样可以借鉴上一节的做法，在 Model 目录新建一个 `index.js` 文件，自动读取所有 Model。

假设我们想要修改某个表中的符合条件的数据，代码可以这样写：

```js
let models = require('models')
models.table.update(
    {
        key: 'value'
    },
    {
        where: {
            condition: 'value'
        }
    }
)
.then(rows => {
    // rows 表示受影响行数
})
```

具体的语法可以参考官方文档，比较坑爹的是，MySQL 执行更新操作只会返回影响的行数，但并不能取到更新后的值。这是 MySQL 自身的限制，其他的数据库可能会支持。

### CAS 登录

既然是后端服务，自然离不开账号体系。一般大一些的公司都会做自己的 SSO（Single Site On），也就是单点登录。

注意：**SSO 只是一种定义，或者说规范，并不是某种实现**，换句话说你只要在 `a.xxx.com` 登陆过，就能确保在 `b.xxx.com` 免登录，就算是实现 SSO 了。

既然是规范，SSO 的实现方式就有很多种，一种比较常见的方式是 CAS（Central Authentication Service），中文可能叫做中心化验证服务。网上很多教程写的天花乱坠，乱七八糟，这里推荐一篇比较通俗易懂的：[前端需要了解的 SSO 与 CAS 知识](https://juejin.im/post/5a002b536fb9a045132a1727)

假设我们要做 `xxx.com` 域名下的 SSO，我们把域名 `a.xxx.com` 叫做 A 站，对应的 B 站就表示 `b.xxx.com`，域名 `cas.xxx.com` 则是一个中心化的服务，用来处理登陆问题，简称 CAS 站，整个流程可以概括如下：

1. 用户首次访问 A 站，A 站服务端发现 cookie 里面没有 SessionID，判定用户未登录
2. 返回状态码 302，重定向到 CAS 站，并且把当前的地址作为参数传递过去，以便未来登陆以后再跳转回来。
3. CAS 站检查 `cas.xxx.com` 域名下的 cookie，发现没有 cookie，判定用户没有在 CAS 登录，提供扫码等方式登录。
4. 登陆成功后会存储 cookie，并且根据请求参数，再返回 302 状态码跳转回 A 站，同时在请求的最后添加一个参数 ticket。这个 ticket 是**用户在 CAS 站登录成功的凭证**，并且有一个很短（比较几秒钟）的生命周期
5. 此时用户会被重定向到 `a.xxx.com/ticket=xxxxxxx` 这个地址，服务端发现了 ticket 参数，向 CAS 站验证 ticket 的有效性，如果有效则将此 SessionId 标记为已登录。

因为 A 站的 SessionId 只有服务端向 CAS 站验证过有效性后，才会标记为已登录，所以如果用户不在 CAS 站登录，再次访问 A 站，虽然 SessionId 存在，但依然不会被识别为已登录。

当用户首次访问 B 站时，流程如下：

1. 用户首次访问 B 站，B 站服务端发现 B 站的 cookie 里面没有 SessionId，判定用户未在 B 站登录。
2. 返回状态码 302，重定向到 CAS 站，并且把当前的地址作为参数传递过去，以便未来登陆以后再跳转回来。
3. CAS 站检查 `cas.xxx.com` 域名下的 cookie，因为曾经登陆过，所以留下了 cookie，判定用户已经登录，直接重定向回 B 站，同时在请求的最后添加一个参数 ticket。
4. 用户携带 ticket 重定向回 B 站，服务端向 CAS 站验证 ticket 有效性，登录成功。

经过这样的梳理，我们会发现访问 B 站的 1、2步和访问 A 站时时一模一样的，3、4 步这和访问 A 站时的 4、5 步一模一样。唯一的区别就是省略了第三步，因为 CAS 站读取到 cookie，免去了登录这一步。而用户虽然经历了两次 302 重定向（B -> CAS -> B），但因为中间没有打断，所以完全是无感知的。

了解完 CAS 登录，我们会发现如果这套东西都由自己写，工作量还是挺高的，要读写 session、重定向、验证 ticket 等。好在 CAS **作为实现而不是规范**，是提供了成熟的工具的，包括服务端和客户端。对于客户端来说，所有需要的参数就是 CAS 站的地址，以及当前站点的地址，其他一切细节都可以通过 CAS 中间件来完成。

### 路由白名单

当有了 CAS 校验以后，我们的后端服务就变得非常“安全”了，客户端（curl 命令或者 Postman）无法直接调用 API，因为这样的请求没有携带 cookie。但有时候在 Jenkins 等平台上又想要调用我们的服务，此时就可以利用路由的顺序了。

比如除了默认的 `/api` 命名空间外，我们还可以设计一个白名单 api（wlapi），然后把它们的顺序定义成这样：

```js
app.use('/wlapi', wlApiRouter)
app.use(cas.bounce)
app.use(express.static(path.join(__dirname, 'public')))
app.use('/api', apiRouter)
```

这样以 `app.use(cas.bounce)` 为分界线，之前的请求都不会做 CAS 校验，之后的请求都需要校验。

### 配合 History 模式路由改造

在 Vue 部分就介绍过，如果觉得 `/#/path` 这种路径不好看，可以借助浏览器的 History 接口将它改造为 `/path`。原理很简单，就是浏览器开放了某个 api，允许开发者改变地址栏的路径，但不执行真正的跳转。

但如果我直接请求 `https://xxx.com/path`，后端还是会接受到这个请求，并且大概率抛出 404，因为这个路径下的资源并不存在，网页本质上还是个 SPA。解决方案是改造后端接口，把所有找不到的资源都重定向到首页的 `index.html` 上。

后端的改造方式有很多种，比如通过 Nginx 来改造等。在 Express 中，我们可以直接用一个现有的 NPM 模块来接入：

```shell
npm install --save connect-history-api-fallback
```

这个中间件虽然是向找不到资源的请求返回 `index.html`，名字里也有个 `fallback`，但这可不意味着这个中间件就要放到所有中间件的最后，用来兜底所有网络请求。

根据官方文档说明，我们可以使用 `verbose` 选项来查看具体的工作流程：

```js
app.use(history({
  verbose: true
}));
```

当我们请求 `/path1` 这个路径时，会看到如下输出：

> Rewriting GET /path1 to /index.html

那么这个中间件是怎么知道是否需要把资源重定向的呢，会不会正常的 api 请求也被重定向了？文档中有如下说明，只有满足以下所有条件才会被重定向：

1. 必须是 GET 请求
2. 请求的 HEADER 中，accept 字段必须包含 `text/html`
3. 不是直接的文件请求，也就是说请求路径不含有 `.`，比如请求 `xxx.com/path/file.json` 就不会被重定向。

其中前两条是硬性要求，说白了就是必须是浏览器发起的 HTML 资源请求，正常的 api 调用不会受到影响。第三条是默认配置，可以在初始化中间件时自行修改。

了解了这个中间件的运行原理之后，很明显在使用中间件以后，还必须指定静态资源的路径，否则一样无法找到 HTML 文件，具体做法可以查看官方文档：[在 Express 中提供静态文件](http://expressjs.com/zh-cn/starter/static-files.html)

## 经验与计划

虽然在项目构建之初就考虑到可维护、快速试错、环境隔离和解耦，但毕竟经验不足。回望整个项目的开发过程，以下几点是做得不够好或者还没有开始做，有待于未来优化的。

### 服务抽取

在业务比较简单的情况下，我们只要在每个 api 对应的 JS 文件里面写业务逻辑就可以了。随着业务逐渐变复杂，有些需求是重复的，这时候就涉及到代码复用的问题了。

有些涉及到和外部系统打交道的事情，比如调用 IM 工具的接口发送消息，调用 Jenkins 启动任务，以及写入数据库等等，我们都可以把它抽象为某个服务。

这样在接口对应的 js 文件里面，我们只需要处理和此接口强耦合的逻辑，然后调用服务即可。尤其是数据库的相关操作，虽然有了 ORM 这样的神器，但我还是强烈建议再用服务来封装一下。首先是可以节约代码量，以更新为例，如果手动写，我们每次都要提供 `where` 字段和更新的字段，但完全可以封装成一个对象，这样以后调用只要传递对象进来就可以了。

当然，更重要的是数据库的操作不能太频繁，很多时候我们需要在数据库前面加一层 Redis。这样命中缓存时就不用去数据库读写了，但同时也需要手动管理缓存的增删和失效。比如在写入、删除、修改数据库时，我们需要同时操作 Redis 缓存， 读取时要先从缓存查找，找不到的话返回数据库里的内容并且添加到缓存。这些都是通用的逻辑，如果每次都手写，就会有大量冗余代码，放在服务里面会更合适。

### 数据约定

数据约定是我对已有的代码逻辑唯一不太满意的地方。以 BOOL 值为例，在通过 HTTP 传输时可以用 `true/false` 或者 `1/0` 来表示它，但无论如何，和语言中原生的布尔类型之间还是要做一次转换。如果数据库设计不太合理，往往也需要转换。

因为 JS 是动态类型的语言，如果不注意数据的约定，回头再来看代码，就会被类型弄得手足无措，只能靠代码的写法去猜类型。如果猜错了，还会比较严重的影响开发效率。所以我的建议是数据约定一定要做好，个人比较推荐三明治原则：**只在接收外部数据和向外提供数据时做类型转换，内部逻辑一律使用原生的对象**，举个例子：

1. 接受 GET 请求，立刻把请求参数解析出来，把字符串的 `1/0` 或者 `true/false` 转成 BOOL 类型
2. 内部逻辑中始终使用转换后的类型，也就是 BOOL
3. 如果要进行 IO 操作，只在服务层转换成字符串，传入服务层的还是 BOOL

BOOL 只是一个例子，另外比较常见的还有 JSON。

### 日志

日志一般可以分为两种，无痕和人工。无痕日志指的是使用 `morgan` 这样的中间件，自动记录所有请求相关的信息。人工记录的日志可以使用 `winston` 这个框架。它们都可以在开发模式下输出到终端，在生产模式下输出到文件。

对于云服务来说，存储到本地的日志文件没有任何意义，下次升级就会丢失数据，所以一般需要对接公司统一的日志系统。不过因为我这个项目不算很关键，所以暂时也就没有对接日至系统的动力，日志更多的时候会用于线上调试。


