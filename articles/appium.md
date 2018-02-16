因为业务需求和准备毕设，最近开始研究自动化测试的内容。由于同时要做 iOS、安卓和 Web 测试，我们最终选择了 Appium 这个开源工具并基于它做一些封装，从而能够使用一套公共 API 完成移动端的双端测试。本文主要会基于一些开源代码和个人实践，对 iOS 端的自动化测试原理做一个简单介绍，Android 略有区别但也大致同理。

其实文章没有很长，也没有太多技术含量，驱使我写这篇文章的主要原因是 Google 上能搜到的绝大多数博客都是错的，大多是根据一篇老旧过时的文章抄抄改改。所以真的很想问问这些文章的作者，你真的搞懂 Appium 的原理么？对这一些错误的知识有可能搞懂么？自己不先搞懂，怎么能昧着良心写进博客里？

# Appium

我假设读者完全没有了解过自动化测试以及相关的概念，那么首先就要搞明白 Appium 是什么，大概由几个步骤组成，接下来才是对每个部分的深入了解。

简单来说，Appium 是一个测试工具，可以进行 iOS、Android 和 Web 测试，同时还允许使用多种语言来编写测试用例。那么问题就变成了为什么 Appium 支持多种语言来写测试用例，以及这些测试用例是如何运行在具体的平台(比如 iOS )上的。为了回答这个问题，我们需要把 Appium 分成三个部分来看，分别是 Appium 客户端、Appium 服务端和设备端。既然是自动化测试，那么就先从设备端说起。

# 设备端

如果你按照[官网的教程](http://appium.io/slate/en/tutorial/ios.html?java#native-ios-automation)成功的运行了 iOS 真机测试，你会看到手机上多了一个名为 **WebDriverAgentRunner** 的应用，以后简称 **WDA**，这个应用的作用就是对你的目标 App 进行测试。

好吧，是不是觉得事情有点神奇了？安装了一个别人的 app，居然能唤起你自己的应用，还能执行测试，是不是存在什么黑魔法？

首先需要介绍一下苹果的 UI 自动化测试框架，在 Xcode 7 以前使用了`UI Automation` 框架，利用 JS 脚本去做应用测试。而在 Xcode 7 中苹果提供了新的框架 `UI Testing`，在 Xcode 8 中干脆直接移除了对 `UI Automation` 的支持。所以毫无疑问，在 iOS 9 或者更高的系统版本中，Appium 也是利用了 `UI Testing` 框架来做测试而不是`UI Automation`。

很多程序员应对 `UI Testing` 框架并不陌生，在新建项目的时候就有机会勾选上这个选项，或者后期通过 `Add target` 的方式补上。默认情况下，一个测试用例就是一个 `.m` 文件，模板代码如下: 

```objc
#import <XCTest/XCTest.h>

@interface Test : XCTestCase

@end

@implementation Test

- (void)setUp {
    [super setUp];
    
    // Put setup code here. This method is called before the invocation of each test method in the class.
    
    // In UI tests it is usually best to stop immediately when a failure occurs.
    self.continueAfterFailure = NO;
    // UI tests must launch the application that they test. Doing this in setup will make sure it happens for each test method.
    [[[XCUIApplication alloc] init] launch];

    // In UI tests it’s important to set the initial state - such as interface orientation - required for your tests before they run. The setUp method is a good place to do this.
}

- (void)tearDown {
    // Put teardown code here. This method is called after the invocation of each test method in the class.
    [super tearDown];
}

- (void)testExample {
    // Use recording to get started writing UI tests.
    // Use XCTAssert and related functions to verify your tests produce the correct results.
}
@end
```

可以看到一共只有三个方法， `setUp` 方法中主要做一些测试前的准备，比如这里的 `[[[XCUIApplication alloc] init] launch];` 就创建了一个被测试应用的实例并唤起它。`tearDown` 方法是测试结束后的清理工作。

所有的测试函数都必须以 `test` 开头，比如这里的 `- (void)testExample`。好吧，不得不承认 OC 这们语言还是有缺陷的，缺少了 Annotation 以后就只能用变量名来做标记，这种业务对应的 Java 表示应该是:

```java
@test
public void example() {
    // do some test
}
```

言归正传，有了这样的测试代码后，只要 `Command + U` 就可以运行测试。不过这还是没有解决之前的疑惑，为什么 Appium 可以用一个第三方 app 唤起待测试的应用并进行调试？当然，这个问题等价于，上面代码中的 `[[XCUIApplication alloc] init]` 到底会创建一个什么样的 app 实例？它怎么知道这个对象的 `launch` 方法会打开手机上的哪个 app？

这个问题似乎没有搜到比较明确的答案，不过经过实践分析以后发现，`XCUIApplication` 类存在一个私有方法，可以传入目标应用的 BundleID:

```objc
[[XCUIApplication alloc] initPrivateWithPath:nil bundleID:@"com.bestswifte.targetapp"];
```

我们知道手机上一个 BundleID 唯一对应了一个应用，后装的应用会替换掉之前相同 ID 的应用，所以通过 BundleID 总是可以正确的唤起待测试应用。唯一要注意的是，为了顺利通过编译器的语法检测，我们在调用私有方法之前需要先构造一份 `XCUIApplication` 的头文件，声明一下将要调用的私有方法。

当我们用这个私有的初始化方法替换掉默认的 `init` 方法后，就可以正常唤起待测试应用了，不过你会发现被测试的应用刚一打开就会退出，这是因为我们的测试代码内容为空，所以很快就会进入到销毁流程。

解决问题也很简单，我们可以在 `testExample` 里面跑一个死循环，模拟 Runloop 的操作。只不过这次不是监听用户事件，而是监听某个 TCP 端口，等待网络传输过来的消息。

我们以 Facebook 开源的 [WDA](https://github.com/facebook/WebDriverAgent) 为例，看看它的 `FBScreenshotCommands.m` 文件:

```objc
#import "FBScreenshotCommands.h"

#import "XCUIDevice+FBHelpers.h"

@implementation FBScreenshotCommands

#pragma mark - <FBCommandHandler>

+ (NSArray *)routes
{
  return
  @[
    [[FBRoute GET:@"/screenshot"].withoutSession respondWithTarget:self action:@selector(handleGetScreenshot:)],
    [[FBRoute GET:@"/screenshot"] respondWithTarget:self action:@selector(handleGetScreenshot:)],
  ];
}


#pragma mark - Commands

+ (id<FBResponsePayload>)handleGetScreenshot:(FBRouteRequest *)request
{
  NSString *screenshot = [[XCUIDevice sharedDevice].fb_screenshot base64EncodedStringWithOptions:NSDataBase64Encoding64CharacterLineLength];
  return FBResponseWithObject(screenshot);
}

@end
```

这里首先会注册一个 `/screenshot` 的路由，并且制定处理函数为 `handleGetScreenshot`，然后函数内部调用 `XCUIDevice` 的截图方法。

所以分析到这里，WDA 的套路就很清晰了，它能根据被测试应用的 BundleID 将它唤起，然后自己进入死循环保证测试用例一直不退出。此时等待服务器传来 URL 数据，然后解析 URL，分发到对应模块，各个模块根据 URL 指令执行对应的测试操作，最终再把测试结果返回。

# Appium 服务端

简单来说，Appium 服务端是一个 Node.js 应用，这个应用跑在电脑上，用于和 WDA 进行通信。刚刚我们看到了截图命令的 URL 是 `/screenshot`，可以想见还有别的类似的测试操作，所以 WDA 有必要和 Appium 服务端约定一套通信协议。考虑到 Appium 还支持 Android 测试，所以在安卓手机上也有类似的东西需要和 Appium 服务端进行交互。这样一来，约定一套通用的协议就显得非常重要。

Appium 采用的是 WebDriver 协议。在 [w3.org](https://www.w3.org/TR/webdriver/) 上有一个对该协议的详细描述，而 [Selenim 的官网](http://www.seleniumhq.org/docs/03_webdriver.jsp) 也介绍了 WebDriver 协议。目前我尚不知道这两处介绍的关系，是互为补充 or 两套规范，但可以肯定的是下面这段话的介绍:

> WebDriver’s goal is to supply a well-designed object-oriented API that provides improved support for modern advanced web-app testing problems.

所以简单的把 WebDriver 理解成一套通用的测试协议即可。

# Appium 客户端

Appium 客户端就是指我们写的那些测试代码了。Appium 支持多种测试语言的根本原因在于，WebDriver 协议为各种主流语言提供了一个第三方库，能够方便的把测试脚本转化成符合 WebDriver 规范的 URL。比如 [https://www.w3.org/TR/webdriver/#list-of-endpoints](https://www.w3.org/TR/webdriver/#list-of-endpoints) 就规定了包括截图、寻找元素、点击元素等操作对应的 URL 格式。

我们的项目目前使用 Java 语言来编写测试脚本，这样的好处是 Android 工程师就可以承担起维护和编写 Android、iOS 两个平台下测试代码的重任了。

# 总结

其实 Appium 的原理非常简单，一句话就能概括:

> 提供各个语言的第三方库，将测试脚本转化成 WebDriver 协议下的 URL，通过 Node 服务发送到各个平台上的代理工具，代理工具在运行过程中不断接收 URL，根据 WebDriver 协议解析出要执行的操作，然后调用各个平台上的原生测试框架完成测试，再将测试结果返回给 Node 服务器。

最后再友情提醒一句:

> iOS 上的真机测试环境很难配置，如果一天搞不定，不要气馁，多试几次，也许明天就好了

# 参考资料:

1. [[iOS9 UIAutomation] What is Appium approach to UIAutomation deprecation by Apple](https://discuss.appium.io/t/ios9-uiautomation-what-is-appium-approach-to-uiautomation-deprecation-by-apple/7319)
2. [现在开始把UI Testing用起来！](http://www.jianshu.com/p/31367c97c67d)
3. [A Beginner’s Guide to Automated UI Testing in iOS](http://www.appcoda.com/automated-ui-test/)

学习的过程中走了不少弯路，如果你看到下面这两篇文章，建议立刻按下 `Command + w`，不要浪费时间去学习错误知识了:

1. [How Appium Works?](http://executeautomation.com/blog/how-appium-works/)
2. [Appium简介及工作原理](http://blog.sina.com.cn/s/blog_c2c7d41b0102xd47.html)
3. [用实例告诉你，如何利用Appium实现移动终端UI自动化测试](http://rdc.hundsun.com/portal/article/584.mhtml)