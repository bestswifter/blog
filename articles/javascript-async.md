本文的例子用 JavaScript 语法给出，希望读者至少有使用过 Promise 的经验，如果用过 async/await 则更好，对于客户端的开发者，我相信语法不是阅读的瓶颈，思维才是，因此也可以了解一下异步编程模型的演变过程。

# 异步编程入门

## CPS 

CPS 的全称是 (Continuation-Passing Style)，这个名词听上去比较高大上(背后涉及到很多数学方面的东西)，实际上如果只是想了解什么是 CPS 的话，并不是太难。

我们看下面这段代码，你肯定会觉得太简单了:

```js
function sum(a, b) {
    return a + b;
}

int a = sum(1, 2);   // 第一行业务代码 
console.log(a);   // 第二行业务代码 
```

隐藏在这两行代码背后的是串行编程的思想，也就是说第一行代码执行出结果以后才会执行第二行代码。

可如果 `sum` 这个函数耗时比较久怎么办呢，一般我们不会选择等待它执行完，而是提供一个回调，在执行完耗时操作以后再执行回调，同时避免阻塞主线程:

```js
function asum(a, b, callback) {
	const r = a + b;
	setTimeout(function () {
		callback(r);
	}, 0);
}

asum(1, 2, r => console.log(r));
```

于是，业务方不用等待 `asum` 的返回结果了，现在它只要提供一个回调函数。这种写法就叫做 CPS。

CPS 可以总结为一个很重要的思想: **“我不用等执行结果，我先假设结果已经有了，然后描述一下如何利用这个结果，至于调用的时机，由结果提供方负责管理”**。

## 没什么卵用的 CPS

扯了这么多 CPS，其实我想说的是，很多介绍 Promise 的文章上来就谈 CPS，更有甚者直接聊起了 CPS 的背后数学模型。实际上 CPS 对异步编程没什么卵用，主要是它的概念太普遍，太容易理解了，我敢打赌几乎所有的开发者都或多或少的用过 CPS。

毕竟回调函调每个人都用过，只不过你不一定知道这是 CPS 而已。比如随便举一个 AFNetworking 中的例子:

```objc
NSURLSessionDataTask *dataTask = [manager dataTaskWithRequest:request completionHandler:^(NSURLResponse *response, id responseObject, NSError *error) {
    if (error) {
        NSLog(@"Error: %@", error);
    } else {
        NSLog(@"%@ %@", response, responseObject);
    }
}];
```

# Promise

写过 JavaScript 的人应该都接触过 Promise，首先明确一个概念，Promise 是一些列规范的总称，现有的规范有 Promise/A、Promise/B、Promise/A+ 等等，每个规范都有自己的实现，当然也可以自己提供一个实现，只要能满足规范中的描述即可。

写过 Promise 或者 RAC/RxSwift 的读者估计对一长串 `then` 方法记忆深刻，不知道大家是否思考过，为什么会设计这种链式写法呢？

我当然不想听到什么“方法调用以后还返回自己”这种废话，要能反复调用 then 方法必然要返回同一个类的对象啊。。。要想搞清楚为什么要这么设计，或者为什么可以这么设计，我们先来看看传统的 CPS(基于回调) 写法如何处理嵌套的异步事件。

如果我需要请求第一个接口，并且用这个接口返回的数据请求下一个接口，那代码看起来大概是这样的:

```js
request(url1, parms1, response => {
    // 处理 response
    request(url2, params2, response => {
        // 处理第二个接口的数据
    })
})
```

上述代码用伪代码写起来看上去还能接受，不过可以参考 OC 的繁琐代码，试想一个双层嵌套就已经如此麻烦， 三层嵌套该怎么写是好呢？

## CPS 的本质

我们抽象一下上面的逻辑，CPS 的含义是不直接等待异步数据返回，而是传入一个回调函数来处理未来的数据。换句话讲:

**回调事件是一个普通事件，内部可能还会发起一个异步事件**

这种世界观的好处在于，通过事件的嵌套形成了一套递归模型，理论上能够解决任意多层的嵌套。当然缺点也是显而易见的，**语义上的嵌套最终导致了代码上的嵌套**，影响了可读性和可维护性。

这种嵌套模型可以用下面这幅图来表示:

![CPS 回调的本质](http://images.bestswifter.com/1488098032.png)

可以看到图中只有两种图形，椭圆形表示一般性事件(回调也是一个事件)，而圆角矩形表示一个异步过程，当执行完以后，就会接着执行它连接着的事件。

## Promise 的本质

当然，我们是有办法解决嵌套问题的，俗话说得好:

> 任何计算机问题都可以通过添加一个中间层来解决

而 Promise 的本质则是下面这幅图:

![Promise 的本质](http://images.bestswifter.com/1488098234.png)

可以看到，我们引入了新的 Promise 层，一个 Promise 内部封装了异步过程，和异步过程结束以后的回调。如果这个回调的内部可以生成一个新的 Promise。于是嵌套模型就变成了链式模型，这也是为什么我们经常能看到 `then` 方法的调用链。

需要强调的是，即使你用了 Promise，也可以在回调函数中直接执行异步过程，这样就回到了嵌套模型。所以 Promise 的精髓实际上在于回调函数中返回一个新的 Promise 对象。

## Promise 的基本概念

数据结构学得好的读者看到上面这幅图应该会想到链表。不过一个 Promise 内部可以持有多个新的 Promise，所以采用的不是链表结构而是有些类似于多叉树。简化版的 Promise 定义如下:

```js
function Promise(resolver) {
  this.state = PENDING;
  this.value = void 0;
  this.queue = [];   // 持有接下来要执行的 promise
  if (resolver !== INTERNAL) {
    safelyResolveThen(this, resolver);
  }
}
```

对一个 Promise 对象调用 `then` 方法，实际上是判断 Promise 的状态是否还是 `PENDING`，如果是的话就生成一个新的 Promise 保存在数组中。否则直接执行 then 方法参数中 block。

当一个 Promise 内部执行完以后，比如说是进入了 `FULLFILLED` 状态，就会遍历自己持有的所有的 Promise 并告诉他们也去执行 `resolve` 方法，进入 `REJECTED` 状态也是同理。

如果能够理解这层思想，你就可以理解为什么有前后关系顺序的几个异步事件可以用 `then` 这种同步写法串联了。因为调用 `then` 实际上是预先保留了一个回调，只有当上一个 Promise 结束以后才会通知到下一个 Promise。

## Promise 小细节

关于 Promise 的实现原理，这篇文章不想描述太多，感兴趣的读者可以参考 [深入 Promise(一)——Promise 实现详解](https://zhuanlan.zhihu.com/p/25178630)，读完以后可以看一下作者的后续文章中的四个题目，检验一下是否真的理解了: [深入 Promise(二)——进击的 Promise](https://zhuanlan.zhihu.com/p/25198178)。

这里我只想强调一下几个容易理解错的地方。首先，Promise 会接受一个函数作为自己的参数，也就是下面代码中的 `fucntion (resolve, reject){ /* do something */ }`:

```js
var p =	new Promise(function (resolve, reject) {
	resolve('hello');
});
console.log(ppppp);
// 打印出 Promise { 'hello' } 而不是 Promise { 'pedding' }
// 证明 Promise 已经在创建时就决议
```

在创建 Promise 时，这个参数函数就会被执行， 执行这个函数需要两个参数 `resoleve` 和 `reject`，它并不是通过 `then` 方法提供而是由 Promise 在内部自己提供，换句话说这两个参数是已知的。

因此如果按照上述代码来写， 在创建 Promise 时就会立刻调用 `resolve('hello')`，然后把状态标记为 `FULLFILLED` 并且让内部的 `value` 值为 `"hello"`。这样后来执行 `then` 的时候会判断到 Promise 已经决议，直接把 `value` 的值放到 then 的闭包中，而且这个过程是异步执行(参考文章中 immediate 的使用)。

有的文章会谈到 Promise 的错误处理，实际上这里没有什么高深的学问或者黑科技。如果在 Promise 内部调用 `setTimeout` 异步的抛出错误，外面还是接不到。

Promise 处理错误的原则是提供了一个 `reject` 回调，并且用 `reject` 方法来代替抛出错误的做法。这样做相当于约定了一套错误协议，把错误直接转嫁到业务方的逻辑中。

另一个需要重点理解的是 `then` 方法提供的闭包中，返回的内容，因为这才是链式模型的核心。

在 Promise 内部的 `doResolve` 方法中会有以下关键判断:

```js
var then = getThen(value);
if (then) {
    safelyResolveThen(self, then);
} else {
    self.state = FULFILLED;
    self.value = value;
    self.queue.forEach(function (queueItem) {
    queueItem.callFulfilled(value);
    });
}
```

因此如果这里的 value 不是基本类型，就会重新走一遍 `safelyResolveThen`，相当于重新解一遍 Promise 了。

所以正确的异步嵌套逻辑应该是: 

```js
var p =	new Promise(function (resolve, reject) {
	resolve('hello');
})
p.then(value => {
	console.log(value);
	return new Promise(function (resolve, reject) {
		resolve('world')
	});
}).then(value => {
	console.log(value);
});

// 第一行打印出 hello
// 第二行打印出 world
```

# 生成器 Generator

我们先看一个 Python 中的例子，如何打印斐波那契数列的前五个元素:

```python
def fab(max): 
    n, a, b = 0, 0, 1 
    while n < max: 
        print b 
        a, b = b, a + b 
        n = n + 1
```

得益于 Python 简洁的语法，函数实现仅用了六行代码:

![fab 函数](http://images.bestswifter.com/1488103781.png)

不过缺点在于， 每次调用函数都会打印所有数字，不能实现按需打印:

```python
for n in fab(5): 
    print n
```

我们先不考虑为什么 `fab(5)` 能放在 `in` 关键字后面，至少能分次打印就意味着我们需要一个对象，内部保存上一次的结果，这样才能正确的生成下一个值。

感兴趣的读者可以用对象来实现一下上述需求， 并且对比一下引入对象后带来的复杂度增加。一种既不增加复杂度，也能保留上下文的技术是使用生成器，只需要修改一个单词即可:

```python
def fab(max): 
    n, a, b = 0, 0, 1 
    while n < max: 
        yield b  #原来是 print b
        a, b = b, a + b 
        n = n + 1 
```

`yield` 关键字的含义是 **当外界调用 next 方法时生成器内部开始执行，直到遇到 yield 关键字，此时把 yield 后面的值传递出去作为 next() 的结果，然后继续执行函数，直到再次遇到 yield 方法时暂停**。

## Generator in JavaScript

上面举 Python 的例子是因为生成器在 Python 中最为简单，最好理解。在 JavaScript 中，生成器的概念稍微复杂一点，主要涉及两个变化。

1. 要求在 function 后面加上星号(*) 表示这是一个生成器而不是普通函数。
2. `next()` 方法可以传递参数，在生成器内部表现为 yield 的返回值。

举个例子:

```js
function* generator(count) {
    console.log(count);
	const result = yield 100
	console.log(result + count);
}

const g = generator(2);  // 什么都不输出
console.log(g.next().value);  // 第一次打印 2，随后打印 100
g.next(9); // 打印 11
```

逐行解释一下:

1. 调用 `generator` 时，生成器并没有执行，所以什么都没有输出。
2. 调用 `g.next` 时，函数开始执行，打印 `2`，遇到 yield，拿到了 yield 生成的内容，也就是 100，传递给 `next()` 的调用结果，所以第二行打印 100。
3. 再次调用 `next()` 方法，生成器内部恢复执行，由于 `next()` 方法传入参数 9，所以 `result` 的值是 9，第三行打印 11。

可见 JavaScript 中的生成器通过 `yield value` 和`next(value)` 实现了值的内外双向传递。

## Generator 的实现

我不知道 Generator 在 JavaScript 和 Python 中的实现原理，然而用 Objective-C 确实可以模拟出来。考虑到生成器内部 **运行 -> 等待 -> 恢复运行** 的特点，信号量是最佳的实现方案。

`yield` 实际上就是信号量的 `wait` 方法，而 `next()` 实际上就是信号量的 `signal` 方法。当然还要处理好数据的交互问题。总的来说思路还是比较清晰的。

# Async/Await

我们先举一个例子，看一下 Promise 的使用，每次调用函数 `p()` 都会生成一个新的 Promise 对象，内部的操作是把参数加一并返回，不妨把函数 p 想象成某个耗时操作。 

```js
function p(t) {
	return new Promise(function (resolve, reject) {
		setTimeout(function () {
			resolve(t + 1);
		}, t);
	});
}
```

假设我需要反复的、线性的执行这个耗时操作，代码将是这样的: 

```js
p(0).then( r => {
	console.log(r);
	return p(r);
}).then( r => {
	console.log(r);
	return p(r);
}).then( r => {
	console.log(r);
	return p(r);
});
```

可见我们调用三次 `then` 方法，执行了三次加一操作，因此会有三行输出，分别是 1、2、3。

## 回调改为线性

文章的一开头就说了，代码总是线性执行， 遇到异步操作不会进行等待，而是直接设置好回调函数并继续向后执行。

实际上，如果借助于 Generator 暂停、恢复的特性，我们可以用同步的方式来写异步代码。比如我们先定义一个生成器 `linear()` 表示内部将要线性执行异步代码:

```js
function* linear() {
	const r1 = yield p(0);
	console.log(r1);
	const r2 = yield p(r1);
	console.log(r2);
	const r3 = yield p(r2);
	console.log(r3);
}
```

我们看到 yield 的值是一个 Promise 对象，为了拿到这个对象，需要调用 `g.next().value`。因此为了让第一个输出打印来，代码是这样的: 

```js
g.next().value.then(value => {  // 其实是 Promise.then 的模式
    // 正如上一节 Generator 的例子中所述，第一个 next 会启动 Generator，并且卡在第一个 yield 上
    // 为了让程序向后执行，还需要再调用一次 next，其中的参数 0 会赋值给 r1。
	g.next(0).value.then()
})
```

如何模拟完整的三个 Promise 调用呢，这要求我们的代码不断向内迭代，同时用一个值保存上一次的结果:

```js
let t = 0;
var g = linear();
g.next().value.then(value => {
	t = value;
	g.next(t).value.then(value => {
		t = value;
		g.next(t).value.then(value => {
			t = value;
			g.next(t)
		})
	})
})
```

这种写法的运行结果和之前用 `then` 语法的运行结果完全一致。

有的读者可能会想问，这种写法完全没有看到好处啊，反而像是回退到了最初的模式，各种嵌套不利于代码阅读和理解。

然而仔细观察这段代码就会发现，嵌套逻辑中更多的是架构逻辑而非业务逻辑，业务逻辑都放在 Promise 内部实现了，因此这里的复杂代码实际上是可以做精简的，它是一个结构高度一致的递归模型。

我们注意到 `g.next().value.then`的内部实际上是重复了外面的调用过程，如何描述这样的递归呢，有一个小技巧，只要在最外层包一个函数，然后递归执行函数就行:

```js
// 递归必然要有可以递归的函数，因此我们在外面包装一层函数
function recursive() {
	g.next(t).value.then(value => {
		t = value;
		return value;
	}).then( result => recursive())
}

recursive();
```

然而有一个问题在于，我们必须在 `recursive()` 函数外面创建生成器 `g`，否则放在函数内部就会导致递归创建新的。因此我们可以加一个内部函数处理核心的递归问题，而外部函数处理生成器和临时变量的创建:

```js
function recursive(generator) {
	let t; // 临时变量，用来存储
	var g = linear();  // 创建整个递归过程中唯一的生成器
	
	function _recursive() {
		g.next(t).value.then(value => {
			t = value;
			return value;
		}).then(() => _recursive())
	}
	_recursive();
}

recursive(linear);
```

可以看到这个 `recursive` 函数完全与业务无关，对于任何生成器函数，比如说叫 g，都可以通过 `recursive(g)` 来进行调用。

这也就通过实际例子简单的证明了即使是异步事件也可以采用同步写法。

需要注明的是，这**并不是** async/await 语法的真正实现，这种写法的问题在于，await 外面的每一层函数都要标注为 async，然而没办法把每一个函数都转换成生成器，然后调用 `recursive()`

感兴趣的同学可以了解一下 [babel 转换前后的代码](https://goo.gl/jlXboV)。

## “同步” 写法的设计哲学

标记了 async 的函数返回结果总是一个 Promise 对象，如果函数内部抛出了异常，就会调用 reject 方法并携带异常信息。否则就会把函数返回值作为 resolve 函数的参数调用。

理解了这一点以后，我们会发现 async/await 其实是**异步操作的向外转移**。

比如说 p 是一个 Promise 对象，我们可能会这样写: 

```js
async function test() {
  var value = await p;
  console.log('value = ' + value);
  return value;
}
test().then(value => console.log(value));
```

我们一定程度上可以把 `test` 当做生成器来看:

1. 调用 test 方法时，首先会执行 test 内部的代码，直到遇到 await。
2. test 方法暂时退出，执行正常的逻辑，此时 test 的返回值尚不可用，但是它是一个 Promise，可以设置 then 回调。
3. await 等待的异步操作结束，test 方法返回，执行 then 回调

因此我们发现异步操作并没有消失，也不可能消失，只是从 `await` 的地方转移到了外面的 `async` 函数上。如果这个函数的返回值有用，那么外部还得使用 `await` 进行等待，并且把方法标记为 `async`。

所以个人建议在使用 `await` 关键字的时候，首先应该判断对异步操作的依赖情况，比如以下场景就非常合适:

```js
async sendRequest(url) {
    const response = await fetch(url);  // 异步请求网络
    const result = await asyncStore(response);  // 得到结果后异步存储数据
}
```

考虑到 `await` 会阻塞执行，如果某个 Promise 后面的代码任然需要执行(比如存储、统计、日志等)，则不建议盲目使用 `await`:

```js
async function test() {
  var s = await fetch(url);
  console.log('这里输出不了啊');
}
```
 
# 参考资料

1. [Python yield 使用浅析](http://www.ibm.com/developerworks/cn/opensource/os-cn-python-yield/)
2. [JavaScript Promise迷你书（中文版）](http://liubin.org/promises-book/)
3. [36个代码块，带你读懂异常处理的优雅演进](http://mp.weixin.qq.com/s?__biz=MzIwNjQwMzUwMQ==&mid=2247484976&idx=1&sn=af2a8b2cabdef9f9396120ca1dd0eae5&chksm=972364f2a054ede406670bf591e0723655c207994a92f7620d4392b66c467610d4be55feab9d#rd)
4. [深入 Promise(一)——Promise 实现详解](https://zhuanlan.zhihu.com/p/25178630)