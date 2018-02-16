> 本文是直播分享的简单文字整理，直播共分为上、下两部分。第一部分：[优酷](http://v.youku.com/v_show/id_XMTUzNzQzMDU0NA==.html) Or [YouTube](https://youtu.be/hPR67T9mbsY)，第二部分：[优酷](http://v.youku.com/v_show/id_XMTU3OTgyMjQwNA==.html)
> 
> Demo 地址：[KtTableView](https://github.com/bestswifter/MySampleCode/tree/master/KtTableView)

如果你觉得 `UITableViewDelegate` 和 `UITableViewDataSource` 这两个协议中有大量方法每次都是复制粘贴，实现起来大同小异；如果你觉得发起网络请求并解析数据需要一大段代码，加上刷新和加载后简直复杂度爆表，如果你想知道为什么下面的代码可以满足上述所有要求：

![解耦后的VC](http://images.bestswifter.com/KtTableview/code.png)

**系好安全带，上车！**

# MVC

在讨论解耦之前，我们要弄明白 MVC 的核心：控制器（以下简称 C）负责模型（以下简称 M）和视图（以下简称 V）的交互。

这里所说的 M，通常不是一个单独的类，很多情况下它是由多个类构成的一个层。最上层的通常是以 `Model` 结尾的类，它直接被 C 持有。`Model` 类还可以持有两个对象：

1. Item：它是实际存储数据的对象。它可以理解为一个字典，和 V 中的属性一一对应
2. Cache：它可以缓存自己的 Item（如果有很多）

常见的误区：

1. 一般情况下数据的处理会放在 M 而不是 C（C 只做不能复用的事） 
2. 解耦不只是把一段代码拿到外面去。而是关注是否能合并重复代码， 并且有良好的拖展性。

# 原始版

在 C 中，我们创建 `UITableView` 对象，然后将它的数据源和代理设置为自己。也就是自己管理着 UI 逻辑和数据存取的逻辑。在这种架构下，主要存在这些问题：

1. 违背 MVC 模式，现在是 V 持有 C 和 M。
2. C 管理了全部逻辑，耦合太严重。
3. 其实绝大多数 UI 相关都是由 Cell 而不是 `UITableView` 自身完成的。

为了解决这些问题，我们首先弄明白，数据源和代理分别做了那些事。

### 数据源

它有两个必须实现的代理方法：

```objc
- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section;
- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath;
```

简单来说，只要实现了这个两个方法，一个简单的 `UITableView` 对象就算是完成了。

除此以外，它还负责管理 `section` 的数量，标题，某一个 `cell` 的编辑和移动等。

### 代理

代理主要涉及以下几个方面的内容：

1. cell、headerView 等展示前、后的回调。
2. cell、headerView 等的高度，点击事件。

最常用的也是两个方法：

```objc
- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath;
- (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath;
```

提醒：绝大多数代理方法都有一个 `indexPath` 参数

# 优化数据源

最简单的思路是单独把数据源拿出来作为一个对象。

这种写法有一定的解耦作用，同时可以有效减少 C 中的代码量。然而总代码量会上升。我们的目标是减少不必要的代码。

比如获取每一个 `section` 的行数，它的实现逻辑总是高度类似。然而由于数据源的具体实现方式不统一，所以每个数据源都要重新实现一遍。

### SectionObject

首先我们来思考一个问题，数据源作为 M，它持有的 Item 长什么样？答案是一个二维数组，每个元素保存了一个 `section` 所需要的全部信息。因此除了有自己的数组（给cell用）外，还有 section 的标题等，我们把这样的元素命名为 `SectionObject`：

```objc
@interface KtTableViewSectionObject : NSObject

@property (nonatomic, copy) NSString *headerTitle; // UITableDataSource 协议中的 titleForHeaderInSection 方法可能会用到
@property (nonatomic, copy) NSString *footerTitle; // UITableDataSource 协议中的 titleForFooterInSection 方法可能会用到

@property (nonatomic, retain) NSMutableArray *items;

- (instancetype)initWithItemArray:(NSMutableArray *)items;

@end
```

### Item

其中的 `items` 数组，应该存储了每个 cell 所需要的 `Item`，考虑到 `Cell` 的特点，基类的 `BaseItem` 可以设计成这样：

```objc
@interface KtTableViewBaseItem : NSObject

@property (nonatomic, retain) NSString *itemIdentifier;
@property (nonatomic, retain) UIImage *itemImage;
@property (nonatomic, retain) NSString *itemTitle;
@property (nonatomic, retain) NSString *itemSubtitle;
@property (nonatomic, retain) UIImage *itemAccessoryImage;

- (instancetype)initWithImage:(UIImage *)image Title:(NSString *)title SubTitle:(NSString *)subTitle AccessoryImage:(UIImage *)accessoryImage;

@end
```

### 父类实现代码

规定好了统一的数据存储格式以后，我们就可以考虑在基类中完成某些方法了。以 `- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section` 方法为例，它可以这样实现：

```objc
- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {
    if (self.sections.count > section) {
        KtTableViewSectionObject *sectionObject = [self.sections objectAtIndex:section];
        return sectionObject.items.count;
    }
    return 0;
}
```

比较困难的是创建 `cell`，因为我们不知道 `cell` 的类型，自然也就无法调用 `alloc` 方法。除此以外，`cell` 除了创建，还需要设置 UI，这些都是数据源不应该做的事。

这两个问题的解决方案如下：

1. 定义一个协议，父类返回基类 `Cell`，子类视情况返回合适的类型。
2. 为 `Cell` 添加一个 `setObject` 方法，用于解析 Item 并更新 UI。

### 优势

经过这一番折腾，好处是相当明显的：

1. 子类的数据源只需要实现 `cellClassForObject` 方法即可。原来的数据源方法已经在父类中被统一实现了。
2. 每一个 Cell 只要写好自己的 `setObject` 方法，然后坐等自己被创建，被调用这个方法即可。
3. 子类通过 `objectForRowAtIndexPath` 方法可以快速获取 item，不用重写。

对照 demo（SHA-1：6475496），感受一下效果。

# 优化代理

我们以之前所说的，代理协议中常用的两个方法为例，看看怎么进行优化与解耦。

首先是计算高度，这个逻辑并不一定在 C 完成，由于涉及到 UI，所以由 Cell 负责实现即可。而计算高度的依据就是 Object，所以我们给基类的 Cell 加上一个类方法：

```objc
+ (CGFloat)tableView:(UITableView*)tableView rowHeightForObject:(KtTableViewBaseItem *)object;
```

另外一类问题是以处理点击事件为代表的代理方法， 它们的主要特点是都有 `indexPath` 参数用来表示位置。然而实际在处理过程中，我们并不关系位置，关心的是这个位置上的数据。

因此，我们对代理方法做一层封装，使得 C 调用的方法中都是带有数据参数的。因为这个数据对象可以从数据源拿到，所以我们需要能够在代理方法中获取到数据源对象。

为了实现这一点， 最好的办法就是继承 `UITableView`：

```objc
@protocol KtTableViewDelegate<UITableViewDelegate>

@optional

- (void)didSelectObject:(id)object atIndexPath:(NSIndexPath*)indexPath;
- (UIView *)headerViewForSectionObject:(KtTableViewSectionObject *)sectionObject atSection:(NSInteger)section;

// 将来可以有 cell 的编辑，交换，左滑等回调
// 这个协议继承了UITableViewDelegate ，所以自己做一层中转，VC 依然需要实现某

@end

@interface KtBaseTableView : UITableView<UITableViewDelegate>

@property (nonatomic, assign) id<KtTableViewDataSource> ktDataSource;
@property (nonatomic, assign) id<KtTableViewDelegate> ktDelegate;

@end
```

cell 高度的实现如下，调用数据源的方法获取到数据：

```objc
- (CGFloat)tableView:(UITableView*)tableView heightForRowAtIndexPath:(NSIndexPath*)indexPath {
    id<KtTableViewDataSource> dataSource = (id<KtTableViewDataSource>)tableView.dataSource;
    
    KtTableViewBaseItem *object = [dataSource tableView:tableView objectForRowAtIndexPath:indexPath];
    Class cls = [dataSource tableView:tableView cellClassForObject:object];
    
    return [cls tableView:tableView rowHeightForObject:object];
}
```

### 优势

通过对 `UITableViewDelegate` 的封装（其实主要是通过 `UITableView` 完成），我们获得了以下特性：

1. C 不用关心 Cell 高度了，这个由每个 Cell 类自己负责
2. 如果数据本身存在数据源中，那么在代理协议中它可以被传给 C，免去了 C 重新访问数据源的操作。
3. 如果数据不存在于数据源，那么代理协议的方法会被正常转发（因为自定义的代理协议继承自 `UITableViewDelegate `）

对照 demo（SHA-1：ca9b261），感受一下效果。

# 更加 MVC，更加简洁

在上面的两次封装中，其实我们是把 `UITableView` 持有原生的代理和数据源，改成了 `KtTableView` 持有自定义的代理和数据源。并且默认实现了很多系统的方法。

到目前为止，看上去一切都已经完成了，然而实际上还是存在一些可以改进的地方：

1. 目前仍然不是 MVC 模式！
2. C 的逻辑和实现依然可以进一步简化

基于以上考虑， 我们实现一个 `UIViewController` 的子类，并且把数据源和代理封装到 C 中。

```objc
@interface KtTableViewController : UIViewController<KtTableViewDelegate, KtTableViewControllerDelegate>

@property (nonatomic, strong) KtBaseTableView *tableView;
@property (nonatomic, strong) KtTableViewDataSource *dataSource;
@property (nonatomic, assign) UITableViewStyle tableViewStyle; // 用来创建 tableView

- (instancetype)initWithStyle:(UITableViewStyle)style;

@end
```

为了确保子类创建了数据源，我们把这个方法定义到协议里，并且定义为 `required`。

# 成果与目标

现在我们梳理一下经过改造的 `TableView` 该怎么用：

1. 首先你需要创建一个继承自 `KtTableViewController` 的视图控制器，并且调用它的 `initWithStyle` 方法。
	
	```objc
	KTMainViewController *mainVC = [[KTMainViewController alloc] initWithStyle:UITableViewStylePlain];
	```
2. 在子类 VC 中实现 `createDataSource` 方法，实现数据源的绑定。

	```objc
	- (void)createDataSource {
		self.dataSource = [[KtMainTableViewDataSource alloc] init]; // 这	一步创建了数据源
	}
	```
	
3. 在数据源中，需要指定 cell 的类型。

	```objc
	- (Class)tableView:(UITableView *)tableView cellClassForObject:(KtTableViewBaseItem *)object {
		return [KtMainTableViewCell class];
	}
	```
	
4. 在 Cell 中，需要通过解析数据，来更新 UI 并返回自己的高度。

	```objc
	+ (CGFloat)tableView:(UITableView *)tableView rowHeightForObject:(KtTableViewBaseItem *)object {
		return 60;
	}
	// Demo 中沿用了父类的 setObject 方法。
	```

# 还有什么要优化的

到目前为止，我们实现了对 `UITableView` 以及相关协议、方法的封装，使它更容易使用，避免了很多重复、无意义的代码。

在使用时，我们需要创建一个控制器，一个数据源，一个自定义 Cell，它们正好是基于 MVC 模式的。因此，可以说在封装与解耦方面，我们已经做的相当好了，即使再花大力气，也很难有明显的提高。

但关于 `UITableView` 的讨论远远没有结束，我列出了以下需要解决的问题

1. 在这种设计下，数据的回传不够方便，比如 cell 的给 C 发消息。
2. 下拉刷新与上拉加载如何集成
3. 网络请求的发起，与解析数据如何集成

关于第一个问题，其实是普通的 MVC 模式中 V 和 C 的交互问题，可以在 Cell（或者其他类） 中添加 weak 属性达到直接持有的目的，也可以定义协议。

问题二和三是另一大块话题，网络请求大家都会实现，但如何优雅的集成进框架，保证代码的简单和可拓展，就是一个值得深入思考，研究的问题了。接下来我们就重点讨论网络请求。

# 为何创建网络层

一个 iOS 的网络层框架该如何设计？这是一个非常宽泛，也超出我能力范围之外的问题。业内已有一些优秀的，成熟的思路和解决方案，由于能力，角色所限，我决定从一个普通开发者而不是架构师的角度来说说，一个普通的、简单的网络层该如何设计。我相信再复杂的架构，也是由简单的设计演化而来的。

对于绝大多数小型应用来说，集成 `AFNetworking` 这样的网络请求框架就足以应付 99% 以上的需求了。但是随着项目的扩大，或者用长远的眼光来考虑，直接在 VC 中调用具体的网络框架（下面以 `AFNetworking` 为例），至少存在以下问题：

1. 一旦日后 `AFNetworking` 停止维护，而且我们需要更换网络框架，这个成本将无法想象。所有的 VC 都要改动代码，而且绝大多数改动都是雷同的。
	
	这样的例子真实存在，比如我们的项目中就依然使用早已停止维护的 `ASIHTTPRequest`，可以预见，这个框架迟早要被替换。

2. 现有的框架可能无法实现我们的需求。以 `ASIHTTPRequest` 为例，它的底层用 `NSOperation` 来表示每一个网络请求。众所周知，一个 `NSOperation` 的取消，并不是简单调用 `cancel` 方法就可以的。在不修改源码的前提下，一旦它被放入队列，其实是无法取消的。

3. 有时候我们的需求仅仅是进行网络请求，还会对这个请求进行各种自定义的拓展。比如我们可能要统计请求的发起和结束时间，从而计算网络请求，数据解析的步骤的耗时。有时候，我们希望设计一个通用组件，并且支持由各个业务部门去自定义具体的规则。比如可能不同的部门，会为 HTTP 请求添加不同的头部。

4. 网络请求还有可能有其他广泛需要添加的需求，比如请求失败时的弹窗，请求时的日志记录等等。

参考当前代码（SHA-1:a55ef42）感受一下没有任何网络层时的设计。

# 如何设计网络层

其实解决方案非常简单：

> 所有的计算机问题，都可以通过添加中间层来解决

读者可以自行思考，为什么添加中间层可以解决上述三个问题。

## 三大模块 

对于一个网络框架来说，我认为主要有三个方面值得去设计：

1. 如何请求
2. 如何回调
3. 数据解析

一个完整的网络请求一般由以上三个模块组成，我们逐一分析每个模块实现时的注意事项：

### 发起请求

发起请求时，一般有两种思路，第一种是把所有要配置的参数写到同一个方法中，借用 [与时俱进，HTTP/2下的iOS网络层架构设计](http://www.jianshu.com/p/a9bca62d8dab) 一文中的代码表示：

```objc
+ (void)networkTransferWithURLString:(NSString *)urlString
                       andParameters:(NSDictionary *)parameters
                              isPOST:(BOOL)isPost
                        transferType:(NETWORK_TRANSFER_TYPE)transferType
                   andSuccessHandler:(void (^)(id responseObject))successHandler
                   andFailureHandler:(void (^)(NSError *error))failureHandler {
                   		// 封装AFN
                   }
```

这种写法的好处在于所有参数一目了然，而且简单易用，每次都调用这个方法即可。但是缺点也很明显，随着参数和调用次数的增多，网络请求的代码很快多到爆炸。

另一组方法则是将 API 设置成一个对象，把要传入的参数作为这个对象的属性。在发起请求时，只要设置好对象的相关属性，然后调用一个简单的方法即可。

```objc
@interface DRDBaseAPI : NSObject
@property (nonatomic, copy, nullable) NSString *baseUrl;
@property (nonatomic, copy, nullable) void (^apiCompletionHandler)(_Nonnull id responseObject,  NSError * _Nullable error);

- (void)start;
- (void)cancel;
...

@end
```

根据前文提到的 Model 和 Item 的概念，那么应该可以想到：**这个用于访问网络的 API 对象，其实是作为 Model 的一个属性**。

Model 负责对外暴露必要的属性和方法，而具体的网络请求则由 API 对象完成，同时 Model 也应该持有真正用来存储数据的 Item。

### 如何回调

一次网络请求的返回结果应该是一个 JSON 格式的字符串，通过系统的或者一些开源框架可以将它转换成字典。

接下来我们需要使用 runtime 相关的方法，将字典转换成 Item 对象。

最后，Model 需要将这个 Item 赋值给自己的属性，从而完成整个网络请求。

如果从全局角度来说，我们还需要一个 Model 请求完成的回调，这样 VC 才能有机会做相应的处理。

考虑到 Block 和 Delegate 的优缺点，我们选择用 Block 来完成回调。

### 数据解析

这一部分主要是利用 runtime 将字典转换成 Item，它的实现并不算难，但是如何隐藏好实现细节，使上层业务不用过多关心，则是我们需要考虑的问题。

我们可以定义一个基类的 Item，并且为它定义一个 `parseData` 函数：

```objc
// KtBaseItem.m
- (void)parseData:(NSDictionary *)data {
    // 解析 data 这个字典，为自己的属性赋值
    // 具体的实现请见后面的文章
}
```

## 封装 API 对象

首先，我们封装一个 `KtBaseServerAPI` 对象，这个对象的主要目的有三个：

1. 隔离具体的网络库的实现细节，为上层提供一个稳定的的接口
2. 可以自定义一些属性，比如网络请求的状态，返回的数据等，方便的调用
3. 处理一些公用的逻辑，比如网络耗时统计

具体的实现请参考 Git 提交历史：SHA-1：76487f7

## Model 与 Item

### BaseModel

Model 主要需要负责发起网络请求，并且处理回调，来看一下基类的 Model 如何定义：

```objc
@interface KtBaseModel

// 请求回调
@property (nonatomic, copy) KtModelBlock completionBlock;
//网络请求
@property (nonatomic,retain) KtBaseServerAPI *serverApi;
//网络请求参数
@property (nonatomic,retain) NSDictionary *params;
//请求地址 需要在子类init中初始化
@property (nonatomic,copy)   NSString *address;
//model缓存
@property (retain,nonatomic) KtCache *ktCache;
```

它通过持有 API 对象完成网络请求，可以定制自己的存储逻辑，控制请求方式的选择（长、短链接，JSON或protobuf）。

Model 应该对上层暴露一个非常简单的调用接口，因为假设一个 Model 对应一个 URL，其实每次请求只需要设置好参数，就可以调用合适的方法发起请求了。

由于我们不能预知请求何时结束，所以需要设置请求完成时的回调，这也需要作为 Model 的一个属性。

### BaseItem

基类的 Item 主要是负责 property name 到 json path 的映设，以及 json 数据的解析。最核心的字典转模型实现如下：

```objc
- (void)parseData:(NSDictionary *)data {
    Class cls = [self class];
    while (cls != [KtBaseItem class]) {
        NSDictionary *propertyList = [[KtClassHelper sharedInstance] propertyList:cls];
        for (NSString *key in [propertyList allKeys]) {
            NSString *typeString = [propertyList objectForKey:key];
            NSString* path = [self.jsonDataMap objectForKey:key];
            id value = [data objectAtPath:path];

            [self setfieldName:key fieldClassName:typeString value:value];
        }
        cls = class_getSuperclass(cls);
    }
}
```

完整代码参考 Git 提交历史：SHA-1：77c6392

## 如何使用

在实际使用时，首先要创建子类的 Modle 和 Item。子类的 Model 应该持有 Item 对象，并且在网络请求回调时，将 API 中携带的 JSON 数据赋值给 Item 对象。

这个 JSON 转对象的过程在基类的 Item 中实现，子类的 Item 在创建时，需要指定属性名和 JSON 路径之间的对应关系。

对于上层来说，它需要生成一个 Model 对象，设置好它的路径以及回调，这个回调一般是网络请求返回时 VC 的操作，比如调用 `reloadData` 方法。这时候的 VC 可以确定，网络请求的数据就存在 Model 持有的 Item 对象中。

具体代码参考 Git 提交历史：SHA-1：8981e28

# 下拉刷新

很多应用的 `UITableview` 都具有下拉刷新和上拉加载的功能，在实现这个功能时，我们主要考虑两点：

1. 隐藏底层的实现细节，对外暴露稳定易用的接口
2. Model 和 Item 如何实现

第一点已经是老生常谈，参考 SHA-1 61ba974 就可以看到如何实现一个简单的封装。

重点在于对于 Model 和 Item 的改造。

### ListItem

这个 Item 没有什么别的作用，就是定义了一个属性 `pageNumber`，这是需要与服务端协商的。Model 将会根据这个属性这个属性判断有没有全部加载完。

```objc
// In .h
@interface KtBaseListItem : KtBaseItem

@property (nonatomic, assign) int pageNumber;

@end

// In .m
- (id)initWithData:(NSDictionary *)data {
    if (self = [super initWithData:data]) {
        self.pageNumber = [[NSString stringWithFormat:@"%@", [data objectForKey:@"page_number"]] intValue];
    }
    return self;
}

```

对于 Server 来说，如果每次都返回 `page_number` 无疑是非常低效的，因为每次参数都可能不同，计算总数据量是一项非常耗时的工作。因此在实际使用中，客户端可以和 Server 约定，返回的结果中带有 `isHasNext` 字段。通过这个字段，我们一样可以判断是否加载到最后一页。

### ListModel

它持有一个 `ListItem` 对象， 对外暴露一组加载方法，并且定义了一个协议 `KtBaseListModelProtocol `，这个协议中的方法是请求结束后将要执行的方法。

```objc
@protocol KtBaseListModelProtocol <NSObject>

@required
- (void)refreshRequestDidSuccess;
- (void)loadRequestDidSuccess;
- (void)didLoadLastPage;
- (void)handleAfterRequestFinish; // 请求结束后的操作，刷新tableview或关闭动画等。

@optional
- (void)didLoadFirstPage;

@end

@interface KtBaseListModel : KtBaseModel

@property (nonatomic, strong) KtBaseListItem *listItem;
@property (nonatomic, weak) id<KtBaseListModelProtocol> delegate;
@property (nonatomic, assign) BOOL isRefresh; // 如果为是，表示刷新，否则为加载。

- (void)loadPage:(int)pageNumber;
- (void)loadNextPage;
- (void)loadPreviousPage;

@end
```

实际上，当 Server 端发生数据的增删时，只传 `nextPage` 这个参数是不能满足要求的。两次获取的页面并非完全没有交集，很有可能他们具有重复元素，所以 Model 还应该肩负起去重的任务。为了简化问题，这里就不完整实现了。

### RefreshTableViewController

它实现了 `ListMode ` 中定义的协议，提供了一些通用的方法，而具体的业务逻辑则由子类实现。

```objc
#pragma -mark KtBaseListModelProtocol
- (void)loadRequestDidSuccess {
    [self requestDidSuccess];
}

- (void)refreshRequestDidSuccess {
    [self.dataSource clearAllItems];
    [self requestDidSuccess];
}

- (void)handleAfterRequestFinish {
    [self.tableView stopRefreshingAnimation];
    [self.tableView reloadData];
}

- (void)didLoadLastPage {
    [self.tableView.mj_footer endRefreshingWithNoMoreData];
}

#pragma -mark KtTableViewDelegate
- (void)pullUpToRefreshAction {
    [self.listModel loadNextPage];
}

- (void)pullDownToRefreshAction {
    [self.listModel refresh];
}
```

### 实际使用

在一个 VC 中，它只需要继承 `RefreshTableViewController`，然后实现 `requestDidSuccess` 方法即可。下面展示一下 VC 的完整代码，它超乎寻常的简单：

```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    [self createModel];
    // Do any additional setup after loading the view, typically from a nib.
}

- (void)createModel {
    self.listModel = [[KtMainTableModel alloc] initWithAddress:@"/mooclist.php"];
    self.listModel.delegate = self;
}

- (void)createDataSource {
    self.dataSource = [[KtMainTableViewDataSource alloc] init]; // 这一步创建了数据源
}

- (void)didReceiveMemoryWarning {
    [super didReceiveMemoryWarning];
    // Dispose of any resources that can be recreated.
}

- (void)requestDidSuccess {
    for (KtMainTableBookItem *book in ((KtMainTableModel *)self.listModel).tableViewItem.books) {
        KtTableViewBaseItem *item = [[KtTableViewBaseItem alloc] init];
        item.itemTitle = book.bookTitle;
        [self.dataSource appendItem:item];
    }
}
```

其他的判断，比如请求结束时关闭动画，最后一页提示没有更多数据，下拉刷新和上拉加载触发的方法等公共逻辑已经被父类实现了。

具体代码见 Git 提交历史：SHA-1:0555db2

# 写在结尾

网络请求的设计架构到此就全部结束了，它还有很多值的拓展的地方。还是那句老话，没有通用的架构，只有最适合业务的架构。

我的 Demo 为了方便演示和阅读，通常都是先实现底层的类和方法，然后再由上层调用。但实际上这种做法在实际开发中是不现实的。我们总是在发现大量冗余，无意义的代码后，才开始设计架构。

因此在我看来，真正的架构过程是当业务发生变更（通常是变复杂了）时，我们开始应该思考当前哪些操作是可以省略的（由父类或代理实现），最上层应该以何种方式调用底层的服务。一旦设计好了最上层的调用方式，就可以逐步向底层实现了。

由于作者水平有限，本文的架构并不优秀，希望在深入理解设计模式，积累更多经验后，再与大家分享收获。