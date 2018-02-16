最初接下开屏广告热启动需求时，对于即将踏入一个什么样的深坑，我心里毫无概念。在当时看来，开屏广告的相关代码已经基本实现，我只要额外添加热启动功能就可以，即使算上调研设计、后端联调加上测试的时间，我也只给自己规划了一周多的时间来完成双端的需求。

# 需求

所谓的开屏广告热启动是指，应用程序进入后台后(按 Home 键或者跳转到其他应用)，等待一段时间再回到应用时展示开屏广告。由于操作系统会定时清理不活跃且占内存的应用，所以此时展示开屏广告会让用户以为应用正在重新启动。由于对用户体验伤害小，甚至很多时候几乎可以做到无感知，所以目前很多日活量较高的 app 都实现了开屏广告热启动功能，常见的有微博、头条等。

如果不考虑最短间隔时间，每天热启动次数上限等附加限制，开屏广告热启动的核心需求其实就在于**准确地检测应用切到后台再回到前台**的行为。所谓的准确，指的是不漏掉真正的进入后台，也不误把普通操作当做进入后台。

这个需求看上去非常容易，直接调用系统 API 即可完成，而在实际开发的过程中却遇到了不少坑，我按照平台逐一分析一下，不会有太多的实现细节，主要是聊聊设计和实现思路。

# iOS

先说我最熟悉的，也是相对来说比较容易实现的 iOS 平台。

在实现开屏广告需求时，从设计角度来考虑，由于 `application:didFinishLaunchingWithOptions:` 函数执行结束后会自动发送通知，所以我们只需要监听 `UIApplicationDidFinishLaunchingNotification` 通知即可。在展示广告时，可以使用 `UIView` 直接盖上一张图片。不过考虑到有倒计时按钮，跳过按钮，以及将来有可能支持除了图片以外的其他格式(比如 VR 视频)，所以使用 `UIViewController` 虽然麻烦些，但也不失为一种稳妥的，方便后续拓展维护的做法。

具体做法就不详细描述了，感兴趣的读者可以参考 [无入侵的开屏广告插入方式](http://www.cocoachina.com/ios/20160628/16828.html)。

前文说过热启动需要满足一定条件，比如进入后台和再次回到前台的时间间隔必须大于某个值，否则回到桌面后快速返回应用也会出现开屏广告，带给用户的体验很差。并且这个值最好是做成服务器动态下发，好处是一旦开屏广告的逻辑出现问题，可以把间隔时间设为非常大的值，从而关闭此功能。同样是出于用户体验考虑，每天开屏广告热启动的次数也需要做限制，超出预设次数以后不再展示。

为了管理以上逻辑，并且与原有开屏广告逻辑有效解耦，单独抽离一个 `HotSplashManager` 类就显得很有必要。由于应用的整个生命周期内都有可能展示开屏广告，所以可以考虑设计为单例模式，并且对外统一暴露一个 `- (BOOL)canShowHotSplashAdvertisement` 方法。

不过由于目前项目中没有使用通知，而是与 `application:didFinishLaunchingWithOptions:` 方法强耦合。所以我接手以后的思路也是沿用前人的代码，主要是在 `applicationDidEnterBackground` 函数中通知 `HotSplashManager` 类应用进入后台。

## 锁屏检测

这里的第一个小坑在于锁屏同样会触发 `applicationDidEnterBackground` 函数，而从逻辑上讲，应用锁屏后再解锁并不应该被认为是一种前后台切换，而如果**已经按 Home 键进入后台**，这时候再锁屏/解锁就不应该影响 App **进入了后台再切回前台**的事实，也就是不影响开屏广告的正常展示，这里的逻辑比较绕，需要整理一下逻辑并仔细测试。

检测锁屏和解锁的方法有好几种， 其中有的方法不能完全兼容 iOS 9、10 两大主流版本。最终找到的有效方案是利用 Darwin 层面的通知:

```objc
// 检测锁屏和解锁
CFNotificationCenterAddObserver(CFNotificationCenterGetDarwinNotifyCenter(), //center
                                NULL, // observer
                                displayStatusChanged,
                                CFSTR("com.apple.springboard.lockstate"),
                                NULL, // object
                                CFNotificationSuspensionBehaviorDeliverImmediately);
                                
// 接受通知后的处理
static void displayStatusChanged(CFNotificationCenterRef center,
                                 void *observer,
                                 CFStringRef name,
                                 const void *object,
                                 CFDictionaryRef userInfo) {
    // 每次锁屏和解锁都会发这个通知，第一次是锁屏，第二次是解锁，交替进行
    [[HotSplashManager sharedInstance] lockStateChanged];
}                        
```

如果不是我的使用方式有误，那么理论上来说是拿不到准确的锁屏 or 解锁状态的，只能知道每次解锁或者锁屏都会触发这个通知，并且第一次一定是锁屏，往后依次交替，所以要在自己的 `HotSplashManager` 中管理好屏幕状态。

## 自然日缓存

每天展示次数有上限就意味着展示次数必须被持久化保存在本地，这可以理解为一种特殊的缓存:“仅在一个自然日内有效，跨日自动清空”。考虑到这样的需求并不是开屏广告这个业务独有，所以不妨抽取成一个基础类: `XXXDailyCache`，并且给它一个 `namespace` 的概念来针对不同业务做隔离。

需要强调的是，虽然很多项目都会实现自己的基础缓存类 `XXXCache`，这里我强烈反对使用继承模式，感兴趣的读者可以参考我之前的文章: [从 Swift 的面向协议编程说开去](https://bestswifter.com/pop/#) 一文的倒数第二节: “继承与组合”，说的就是这种非常常见的误用继承关系的场景。所以这里正确的做法是使用组合模式，用 `namespace` 去创建基础的 `XXXCache` 类实现缓存功能，而 `DailyCache` 则持有缓存对象并且实现按自然日删除的逻辑。

按自然日区分的逻辑很简单， 只要把缓存的 Key 设置为当前日期，然后每次读取之前先判断日期即可。这是比较简单的体力活，就不多费口舌了。封装得好的话，只会对外暴露三个简洁方法:

```objc
@interface XXXDailyCache : NSObject

- (id)initWithNameSpace:(NSString *)namespace;
- (id)getValueWithKey(NSString *)key;
- (void)writeWithKey:(NSString *)key value:(id)value;

@end
```

## 开屏广告热启动

之前的同事已经实现了开屏广告功能，他们提供了一个 `showSplashAD` 的方法，方法内部会把根 `UIView` 的 `rootViewController` 设置为开屏广告的 ViewController。

现在添加好了相关判断条件以后，只需要简单改造一下 app 进入前台的回调即可，对原来业务的改动相对来说很小:

```objc
- (void)applicationWillEnterForeground:(UIApplication *)application {
    if ([[HotSplashManager sharedInstance] canShowSplashAd]) {
        [self showSplashAD];
    }
}
```

总的来说 iOS 的实现相当简单， 做好基础类的封装，注意判断一下锁屏问题就可以了。

# Android

首先 iOS 存在的问题安卓都有，所以同样需要封装自然日失效的 `DailyCache`，处理锁屏逻辑则是使用了通知机制，监听系统的通知。

为了可复用性，我们可以封装出一个单独的类来监听锁屏通知，并记录当前状态，以便将来可以对外提供相应的服务:

```java
public class ScreenLockReceiver extends BroadcastReceiver {
    @Override
    public void onReceive(Context context, Intent intent) {
        boolean isScreenOff = false;
        if (intent.getAction().equals(Intent.ACTION_SCREEN_OFF)) {
            isScreenOff = true;
        } else if (intent.getAction().equals(Intent.ACTION_SCREEN_ON)) {
            isScreenOff = false;
        }
    }
}
```

然后实例化这个 `ScreenLockReceiver` 并为它添加好过滤:

```java
ScreenLockReceiver screenLockReceiver = new ScreenLockReceiver();
IntentFilter lockFilter = new IntentFilter();
lockFilter.addAction(Intent.ACTION_SCREEN_ON);
lockFilter.addAction(Intent.ACTION_SCREEN_OFF);
lockFilter.addAction(Intent.ACTION_USER_PRESENT);
registerReceiver(screenLockReceiver, lockFilter);
```

## 前后台切换

由于安卓没有提供系统级别的前后台切换通知，所以不得不自己手动实现。第一种思路是实现 `onTrimMemory` 函数:

```java
@Override
public void onTrimMemory(final int level) {
    if (level == ComponentCallbacks2.TRIM_MEMORY_UI_HIDDEN) {
        // Get called every-time when application went to background.
    }
}
```

它的原理来源于官网对于 `onTrimMemory` 的解释，当 level 的值是 `TRIM_MEMORY_UI_HIDDEN` 时，按照文档的解释是应用程序进入后台，需要释放 UI 资源。基于这种思路的前后台切换检测在 [Stack Overflow](http://stackoverflow.com/a/19920353) 上得到了非常多的赞同。然而根据我们的测试，在某些高端机型上，即使应用程序进入后台，由于内存相对充足，并不会触发上述方法。

考虑到官方文档没有明确说明进入后台时一定会调用 `onTrimMemory` 方法， 很多时候是开发者自己的总结，我们最终放弃了这种实现思路。

实际上还有一种最老土，也相对来说最准确的判断方法。应用进入后台时，Activity 会调用 `onPause` 方法，回到前台又会调用 `onResume` 方法。虽然在切换 Activity 时也会走这样的流程，但是两个方法的调用时间间隔非常短，即使考虑到低端机的性能问题， 两秒钟也足够完成一次页面跳转了。所以只需要记录 `onPause` 的时间戳，再拿到 `onResume` 的时间戳，两者做差比较即可。

如果之前的应用封装的好的话，应该会有一个继承自系统的 `Activity` 的子类，比如叫做 `BaseActivity`。显然以上逻辑应该在这个 `BaseActivity` 里完成， 不过一个应用中并不一定所有的视图都继承自这个 `BaseActivity`，我们还有可能使用 `FragmentActivity` 及其子类，所以在对应的 `BaseFragmentActivity` 中也要添加类似的逻辑。

## 展示开屏广告

与 iOS 不同的是，进入前台事件的直接处理逻辑应该写在 `HotSplashManager` 类中，而非 iOS 的 `Appdelegate`，唤起开屏广告的方式也略有不同。在 `HotSplashManager` 中我们可以直接拿到当前展示的 activity(`BaseActivity` 把自己传过来)，然后调用它的 `startActivity` 方法就可以唤起开屏广告所在的 Activity 了，注意关掉动画效果。

开屏广告的 `SplashActivity` 也需要对唤起方式做区分，判断自己是冷启动展示还是热启动展示。如果是热启动展示，不需要涉及后续的引导页流程，而是直接调用 `this.finish()` 即可。

## 多进程通信

以上功能完成以后，基本上开屏广告热启动的需求就算开发完了，直到测试时有用户反馈全屏查看图片时大概率也会展示开屏广告。经过排查后发现，我们的应用中诸如查看图片、打开网页等操作都会放到其他进程中完成，从而避免与主进程争夺内存，导致 OOM。

多进程场景下会有多个 Application 和 Activity 实例在同时运行。在主进程切换到子进程的过程中，实际上调用到的是主进程的 `onPause` 和子进程的 `onResume`，子进程回到主进程时调用的则是子进程的 `onPause` 和主进程的 `onResume`。

不难看出对于主进程而言，`onPause` 和下一次 `onResume` 之前的时间间隔至少是在子进程中停留的时间。所以容易出现前后台切换的误报。

解决这个问题有多个思路，但任何基于 Application 类，利用内存存储数据的做法均不可能实现，应该避免在这种思路上浪费不必要的时间。首先可以考虑 AIDL、Binder 等多进程通信模型，不过网上搜了一圈，普遍实现起来比较啰嗦，而且实际上我这里的需求并不是通信，而是传递一个非常小的数据，表示 App 是否进入子进程，所以这些方案首先排除。

由于没有找到合适的跨进程内存共享方案，所以接下来考虑的是文件共享的方式，代表技术有 ContentProvider。不过 ContentProvider 实际上是对下层具体文件读写实现方案的抽象封装，提供了一套 CURD 接口。也就是说我还得自己实现文件的读写。考虑到实现成本过大，而需求比较简单，也排除了这种方案。

最后考虑到通知，先调研了 `LocalBroadcastManager` 这种本地通知，看了一下源码以后发现不适用于跨进程通信，内部其实是利用 `context` 参数拿到了 `ApplicationContext`，然后实现了简单的观察者模式。感兴趣的读者可以阅读文章末尾的参考资料。

最终的解决方案是选择了普通的 `BroadcastManager`，注意添加 filter 过滤一下，以及设置好包名，限制广播的接收者。

```java
Intent intent = new Intent(BROADCAST_KEY);
intent.setPackage(getPackageName());
intent.putExtra("flag", false);  // 通知主进程 application: "已经进入子进程"
sendBroadcast(intent);
```

## 其他的一些思考

开发过程中的坑远远不止上面列出的这些，比如还顺手解决了一个弱引用过早释放的 bug 和一个内存泄漏的问题。此外，在开发的过程中对 context 概念还比较模糊，Java 闭包对捕获的变量的处理也挺有意思，不过考虑到大部分都是 Java 语法，就不在这篇业务学习总结里面多啰嗦了，待我整理一下，另起一篇文章专门讨论。

# 参考文章

1. [Android编程之LocalBroadcastManager源码详解](http://blog.csdn.net/xyz_fly/article/details/18970569)
2. [LocalBroadcastManager 的实现原理，还是 Binder？](http://www.trinea.cn/android/localbroadcastmanager-impl/)
3. [LocalBroadcastManager分析](http://www.dundunwen.com/article/765314db-4072-4022-bb21-3f1e64ea54ba.html)
4. [无入侵的开屏广告插入方式](http://www.cocoachina.com/ios/20160628/16828.html)
5. [How to detect when an Android app goes to the background and come back to the foreground](http://stackoverflow.com/questions/4414171/how-to-detect-when-an-android-app-goes-to-the-background-and-come-back-to-the-fo)
6. [Android 探索之 BroadcastReceiver 具体使用以及安全性探究](https://gold.xitu.io/entry/57c3841d0a2b58006cf9f7f5)

以及其他阅读过但没有记录下来的优秀文章，感谢前辈们的分享。