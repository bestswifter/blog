在开始方案介绍之前，请确保自己有一个国外的服务器，比较常用的有以下几个：

1. [Digital Ocean](https://m.do.co/c/0b258b3eea48)：最低价格是 $5/月，每月 1000G 流量。现在注册有 $10 优惠券，GitHub 学生认证以后可以免费领取 $50 优惠券
2. [Vultr](https://www.vultr.com/?ref=7220709)：最低每月 $2.5，500G 流量也足够用，目前没有优惠
3. [搬瓦工](https://bwh88.net/aff.php?aff=19860)：年付 $19，500G 流量，硬盘只有 10G，基本上只能用来科学上网，网上有各种优惠码，大约可以优惠 5% 左右。

如果读者还没有自己的 VPS，可以根据自己的消费能力和使用场景（单纯科学上网还是部署服务）来选购，从上面的链接购买，我也会有奖励。

Shadowsocks 的配置非常简单，参考[这个脚本](https://github.com/nkwsqyyzx/.sys.config/blob/master/init.scripts/install-centos)，只要一行命令，指定端口和密码即可。

## 常见痛点 & 需求：

最优秀的配置就是在使用时感觉不到配置的存在，然而复杂的应用场景决定了我们需要多种代理配置。虽然最终的目的是摆脱繁琐的配置切换，但在此之前，我们有必要梳理一下日常使用中常见的痛点。

### Mac

1. 模拟器需要连代理，因此电脑必须设置 HTTP/HTTPS 代理
2. 终端无法使用 SS 代理，即使设置了全局代理也不行
3. 某些应用，比如 Telegram，需要科学上网

### Mobile

1. 在公司联调时需要手动输入 Mac 的  IP 地址，过程繁琐
2.  如果想科学上网，需要连接 VPN，这与 HTTP 代理冲突
3. 某些网站或应用会对客户端证书做校验， 比如 iTunes、微信支付、公司内网服务等对安全要求较高的场景，导致挂着代理无法访问

### Common

1. 访问墙外网站需要连 VPS，墙内网站不需要连接，代理会让国内网站访问速度降低

## 本质

复杂的需求导致了代理场景的多样化，一般来说有三种：

1. Charles 的 HTTP(s) 代理
2. VPS 的 socks5 代理或 PAC
3. 直接连接

上述问题的本质都来源于这三种代理是互斥的，而且切换起来比较麻烦。在 Mac 上有脚本可以切换，但有时依然无法人工区分改用哪种代理，在 iOS 上，就连设置代理都会非常耗时。

## 目标与解决思路

明确了问题与问题背后的复杂度以后，我们需要确定一个目标，以目标为导向，来指导自己的行动。

目标其实非常简单：

> **经过有限次的配置，彻底避免切换代理**

换句话说，就是希望代理的切换时自动进行的，全程不再需要人工干预。

这似乎有悖于目前的工作模式，不过没关系，既然系统代理不方便，我们就需要一个全局代理，起到 **路由** 作用，根据一定的规则把请求转发到合适的代理上去。这样就把代理的切换简化成了规则的配置，切换永远存在，但配置只要不断维护完善就行。

## Mac 配置

Mac 上用到四个工具：

1. Proxifier： 全局控制工具，负责应用层面的请求转发
2. Charles：一种代理，负责处理那些需要被抓包的请求
3. GoAgentX：一种代理，负责处理那些需要走 Shadowsocks 协议的请求
4. SwitchyOmega：Chrome 拓展工具，负责对 Chrome 中的网页进行智能代理选择

### GoAgentX

这个软件的作用是帮助我们在本地的某个端口（比如 1234）上运行 Shadowsocks 服务，如果某个应用需要科学上网，只要把流量用 socks5 协议代理到 1234 这个端口上即可。

![图片](http://bos.nj.bpc.baidu.com/v1/agroup/1efa16cd2a1701771179f22af014b7e2e07b836b)

有三个比较常用的模式：

1. 全局代理模式：会把系统的 SOCKS 代理设置为 127.0.0.1：1234，这样系统的所有流量都会走 socks 代理
2. 自动代理配置模式（PAC）：这种模式会自动分析 URL，根据 URL 的特点来决定走哪种代理，也就是达到了按需翻墙的目的
3. 独立模式：这种模式下仅仅会在本地的 1234 端口上运行 Shadowsocks 服务，不会更改系统的代理方式，也就是说单独使用无法进行翻墙，需要配合其它的软件来使用，这里我们选择第三种模式。

### Proxifier

[Proxifer](https://www.proxifier.com/) 是一个控制 app 代理方式的软件，有了它，我们就可以控制某个 app，比如 Telegram 使用哪种代理。

首先在 Proxifier 中建立一个 Shadowsocks 代理：

![图片](http://bos.nj.bpc.baidu.com/v1/agroup/ba5025c6036eaa102b2995b867aed89a945eb005)

依次选择 Proxies、Add 并填写本机的端口。

接下来就可以把某些特定应用的流量转到代理上了：

![图片](http://bos.nj.bpc.baidu.com/v1/agroup/8dd329807fce048f888a618874cf743d7d55ec71)

依次选择 Rules、Add、➕，然后选择需要修改的软件以及使用哪种代理。比如这里可以选择强制 Telegram 使用 Shadowsocs 代理，就可以翻墙了。

### 隐藏图标

对于 Proxifier 这类应用来说，他们具有以下特点：

1. 需要开机启动
2. 开机后不会关闭，可以后台运行，并且基本上不再需要人工干预

一般在应用设置（Command + 逗号）中可以设置开机启动，以及隐藏 Menu Bar 中的图标，如果无法设置开机启动，我们可以手动添加：

![图片](http://bos.nj.bpc.baidu.com/v1/agroup/6686f99b8ea4d6a250d1db282da1b11420e437b2)

除了隐藏 Menu Bar 中的图标外，我们还希望把它在 Dock 中的图标也隐藏掉，这需要设置 ` Application is agent` 项，以 Proxifier 为例，需要修改 `/Applications/Proxifier.app/Contents` 和该路径下 `/Applications/Proxifier.app/Contents/Info.plist` 这个文件的权限：

```bash
chmod 777 /Applications/Proxifier.app/Contents
chmod 777 /Applications/Proxifier.app/Contents/Info.plist
```

然后编辑 plist 文件，新增一个布尔类型的选项 **Application is agent (UIElement)**，值为 YES：

![图片](http://bos.nj.bpc.baidu.com/v1/agroup/7a068ff6d96be20ac0681d86b6fc5c1213d6dda1)

这样 Proxifier 就不会在 Dock 中显示，基本上做到了全局无感知。

### 终端科学上网

有些时候，安装一些软件包会非常慢，这是因为数据源存在国外的服务器上，但终端默认不会走代理，即使设置了全局代理也无效。

[为终端设置Shadowsocks代理](http://droidyue.com/blog/2016/04/04/set-shadowsocks-proxy-for-terminal/) 这篇文章介绍了一个工具来解决上述问题：

```bash
brew install polipo
ln -sfv /usr/local/opt/polipo/*.plist ~/Library/LaunchAgents
vim /usr/local/opt/polipo/homebrew.mxcl.polipo.plist
```

添加一行 **<string>socksParentProxy=localhost:1234</string>**，记得把  1234 改成自己的 Shadowsocks 端口号：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
  <dict>
    <key>Label</key>
    <string>homebrew.mxcl.polipo</string>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/opt/polipo/bin/polipo</string>
        <string>socksParentProxy=localhost:1234</string>
    </array>
    <!-- Set `ulimit -n 20480`. The default OS X limit is 256, that's
         not enough for Polipo (displays 'too many files open' errors).
         It seems like you have no reason to lower this limit
         (and unlikely will want to raise it). -->
    <key>SoftResourceLimits</key>
    <dict>
      <key>NumberOfFiles</key>
      <integer>20480</integer>
    </dict>
  </dict>
</plist>
```

接下来向 .zshrc 中加入一个函数：

```bash
function pt() {
    launchctl unload ~/Library/LaunchAgents/homebrew.mxcl.polipo.plist
    launchctl load ~/Library/LaunchAgents/homebrew.mxcl.polipo.plist
    export http_proxy=http://localhost:8123
    export https_proxy=http://localhost:8123
}
```

至此，所有的配置就结束了。如果某个终端窗口需要科学上网，只要先执行 `pt` 函数就行。如果你希望自己的终端永远使用 Shadowsocks 代理，只要将 `pt` 函数中的内容提出来，直接写进 `.zshrc` 文件中即可。

为了测试是否成功的使用了代理，我们可以尝试访问下 Google 或者检测自己的 IP 地址：

![图片](http://bos.nj.bpc.baidu.com/v1/agroup/4b43710762224a8e5345eae45dcf35a148fe0187)

### Charles 

Charles 作为一个大名鼎鼎的抓包工具，具体使用方法就不用我细说了，可以参考巧哥的这篇文章：[iOS开发工具-网络封包分析工具Charles](http://blog.devtang.com/2013/12/11/network-tool-charles-intr/)。

虽然 Charles 的抓包功能足够强大，但美中不足的是，它的数据处理能力极为孱弱。虽然有 Map Local/Remote 和 Rewrite 功能，但终究不是图灵完备的语言， 灵活性和可拓展性还不够强大，比如如果数据是使用 protobuf 的形式传输，Charles 就无能为力了。

通常情况下，为了搭建一个多端统一、长期可维护的测试环境，我们需要一个中间人服务器，经过一段时间的尝试和技术选型，最终我们选择了 MITM，详细内容可以参考我的这篇博客：[HTTP 代理服务器技术选型之旅](http://www.jianshu.com/p/23f77b603748)。

有了 MITM 工具以后，iOS 模拟器的调试问题就迎刃而解了。注意，此时不要直接修改系统的代理，因为来回切换代理比较麻烦，Charles 提供了一个非常好用的功能，叫 External Proxy。顾名思义，Charles 可以把自己视为一个中转站，把经过自己的数据发送到下一个代理服务器上去。

因此，日常工作中，我采用的方案是系统代理长期设置为 Charles，且 Charles 长期将外部代理设置为 MITM，两者要么同时生效，要么同时失效，经过 Charles 数据一定会经过 MITM 。

![图片](http://bos.nj.bpc.baidu.com/v1/agroup/ba67ad53ef4a996c029fcdea86f9cdbaec0faa38)
![图片](http://bos.nj.bpc.baidu.com/v1/agroup/8885cd21d760c8a8f6a3373613338e9512c35446)


### 快速切换系统代理

上述配置虽然大部分时候都能 work，但万一出现 Proxifier 不能指定代理，而且这个网络流量又不能走代理服务器时，我们还需要一个终极方案——快速把系统代理还原为默认。虽然我至今只在某次更新 App Store 时遇到过上述问题，但做好技术储备还是没有坏处的。

使用 `networksetup` 配合各种参数即可实现上述需求，我写了一个可以自动切换 Charles 和 GoAgentX 代理的脚本，在[这里](https://github.com/bestswifter/macbootstrap/blob/master/zsh-config/platform.mac.sh#L8-#L74) 可以下载并根据自己的需求自行修改。

效果如图所示：

![](https://o8ouygf5v.qnssl.com/1506333678.png)

### SwitchyOmega

最后一个要用到的工具是 Chrome 的一个插件：[SwitchyOmega](https://chrome.google.com/webstore/detail/proxy-switchyomega/padekgcemlokbadohgkifijomclgjgif)，它可以给不同的 URL 配置不同的访问策略。

首先我们新建一个情景，用来处理需要科学上网的网站。

![图片](http://bos.nj.bpc.baidu.com/v1/agroup/0477acb8a3718187b023c7931c1eced70435b380)

如果是前端开发，那些需要抓包调试的网站可以代理到 Charles 上，过程类似就不赘述了。

然后需要配置下 **auto switch** 场景，在绝大多数时候，我们都将使用这个场景：

![图片](http://bos.nj.bpc.baidu.com/v1/agroup/e880e213a3f727d1c3f055387be1dc118f53b37a)

第一个区域内是一些自己制定的规则，如果自动配置中某些网站的访问方式不符合我们的要求，可以单独列在这里。第二个区域表示如果命中规则列表，则使用 socks 代理，否则就直接连接。第三个区域则使用了一个开源的网址列表，列出了那些需要科学上网的网站。

有了这样的配置，我们就可以实现 百度、淘宝、京东等网站直接连接，YouTube、Google、草榴等网站科学上网了。

### Mac 配置小结

Mac 上需要用到的软件较多，但每个软件的使用都不太复杂，回忆一下之前的几个痛点：

1. 模拟器需要连代理，因此电脑必须设置 HTTP/HTTPS 代理。这一点我们确实是通过长期设置代理为 Charles -> MITM 实现的，并且利用脚本来快速切换代理。
2. 终端无法使用 SS 代理，即使是全局代理也不行。这个问题通过 `polipo` 工具解决，并且编写了脚本。
3. 某些应用，比如 Telegram，需要科学上网。这个问题通过 Proxifier 来设置某个 app 使用的代理方式。
4. 按需代理，只有需要科学上网的网站才用代理。这个需求通过 Chrome 的插件 SwitchyOmega 完成。

## iOS 配置

对于预算并不充裕的同学来说，小火箭 Shadowrocket 无疑是最佳工具了。这类应用已经在国内的 App Store 下架，需要使用美国账号购买，如果团队内的其它小伙伴也需要，可以把 ipa 用开发者证书或者企业证书重签名下，来实现团队内部分享的目的。

打开 shadowrocket 以后，在首页点击右上角的加号，可以新建一个节点，也就是我们的代理服务器：

 ![图片](http://bos.nj.bpc.baidu.com/v1/agroup/e39526b4e13e4dd99ced278a4799e841f0354538)

这里强烈建议给节点添加备注，原因稍后会讲。

### 自动选择代理

![图片](http://bos.nj.bpc.baidu.com/v1/agroup/dc956533d21907d2dd7dc7bbb4eafd3c1d76b3fb)

接下来，我们需要选择路由模式，一共有四种。直连的意思很好理解，就是不走任何代理，相当于没用上 shadowrocket，可以理解为一种回滚方案。全局则表示 app 所有网络流量都走代理服务器，相当于使用了系统的 VPN。这两者功能都太单一，基本上不会使用。 

配置模式和上文说过的 **auto switch** 差不多，都是实现了黑名单 + 广告屏蔽的功能：

1. 广告屏蔽：对于某些特定的、专门用于投放广告的域名，直接屏蔽掉网络请求，从而起到去除广告的效果。
2. 黑名单：列出需要科学上网的网址，走代理服务器。
3. 其它：默认是直接连接。

我使用的是这份配置：https://raw.githubusercontent.com/lhie1/Surge/master/Shadowrocket.conf

它的主体结构可以分为三部分：

```objc
// 屏蔽广告

// 爱奇艺
DOMAIN-SUFFIX,ad.m.iqiyi.com,REJECT
DOMAIN-SUFFIX,afp.iqiyi.com,REJECT
DOMAIN-SUFFIX,api.cupid.iqiyi.com,REJECT
.......

// 优酷
......

// 科学上网

// 以 gmail.com 结尾的网址要求走代理，并且要求代理服务器做 DNS 解析
DOMAIN-SUFFIX,gmail.com,PROXY,force-remote-dns

// 直连

// BAT 等首页直连
DOMAIN-SUFFIX,baidu.com,DIRECT
DOMAIN-SUFFIX,qq.com,DIRECT
DOMAIN-SUFFIX,alipay.com,DIRECT

```

这套配置绝大多数情况下足够使用了，奈何客户端开发者经常需要对 APP 抓包，以前是通过修改系统代理实现的，现在直接修改配置文件就行了。

### 自定义配置

现在我们的需求变成了：

1. 检查是否是广告，如果是广告则拒绝连接
2. 检查是否需要科学上网，如果需要的话，走代理服务器
3. 检查是否需要抓包，如果需要的话，走 Mac 的  Charles 代理代理代理
4. 剩下的请求默认直连

其中第三四步的逻辑可以颠倒，也就是说可以默认走抓包模式，只有那些不能抓包的连接才直连。然而这种模式问题比较多，因为很多对安全性比较高的 app 或者网页，比如支付模块等等，都会检查客户端证书， 这种情况下通过 Charles 代理就无法访问了。访问了。

这种需求是无法使用默认的配置模式实现的， 因为它只能选择代理 or 直连，而我们的需求实际上是有两种代理，即 Shadowsocks 代理和 Charles 代理，这需要我们对配置文件做一些手动编辑。

注意到之前的每一条配置都是三段式：

> 规则类型》 > 规则类型》 > 规则类型》 > 规则类型,规则特征,连接方式

其中连接方式可以是 DIRECT（直连）、PROXY（代理）和    REJECT（拒绝），实际上这里还可以填写前文所说的节点标签。比如我把 VPS 的标签设置为 DigitalOcean，这里就可以改成 DigitalOcean，效果，效果，效果，效果和 PROXY 一致。

有了这个大前提，我们可以新建一个 HTTP 代理，，，，标记为 Mac，然后把需要抓包的连接的代理方式设置为 Mac 即可：

![图片](http://bos.nj.bpc.baidu.com/v1/agroup/92e2056e5d6fd321fa051e1a30fdc5bd5b9bf2d8)

配置文件更新为：

```
// 需要抓包的地址走 Charles 代理
DOMAIN-SUFFIX,api.example.com,MAC PRO

// 屏蔽广告
DOMAIN-SUFFIX,ad.m.iqiyi.com,REJECT

// 以 gmail.com 结尾的网址要求走代理，并且要求代理服务器做 DNS 解析
DOMAIN-SUFFIX,gmail.com,PROXY,force-remote-dns

// BAT 等首页直连
DOMAIN-SUFFIX,baidu.com,DIRECT
```

### 场景模式 与 Widget

真正让我下定决心使用 shadowrocket 而不是其它应用的原因是它的场景模式。有时候，在 WiFi 模式和 4G 模式，甚至是不同的 WiFi 下，我们使用的配置文件并不相同。比如我在家的 WiFi 和外出 4G 模式下，并不需要抓包。一个场景就是一种数据接入方式和配置文件的集合，最神奇的是，场景可以自动切换。

比如我创建了三个场景，分别表示 4G、家庭 WiFi 和公司 WiFi，每个场景有自己的配置文件，通过 WiFi 名称来区分：

![图片](http://bos.nj.bpc.baidu.com/v1/agroup/a49681daab457fcec22dded470c79a6cb7687b51)

这样当我回家以后，shadowrocket 切换场景，自然也就会切换配置文件，从而避免了手动切换。

### iOS 配置总结

在 iOS 平台上，我们也遵循了不用系统代理，利用全局代理软件来控制代理的原则，通过 shadowrocket 提供的配置文件和规则实现了需求。同时 Widget、场景模式、iCloud 备份与二维码导出提高了软件的使用效率。另一个好用的工具就是 Widget，除了全局开关以外，可以快速在场景、代理、直连、配置四种路由方式中快速选择，这个相当于之前我们编写的切换系统代理的脚本。虽然用到的场景不多，但作为一种备用方案，还是不可或缺的。

![图片](http://bos.nj.bpc.baidu.com/v1/agroup/b2b48b033800ba57c5ca8a01d5204baa2041c3ac)

除此以外，shadowrocket 导入导出配置也很容易，默认使用 iCloud 同步数据，自己的节点和配置还可以用二维码的方式分享给别人，别人在使用时只要扫码即可导入。

## 总结

在 iOS 平台上，我们也遵循了不用系统代理，利用全局代理软件来控制代理的原则，通过 shadowrocket 提供的配置文件和规则实现了需求。同时 Widget、场景模式、iCloud 备份与二维码导出提高了软件的使用效率。

有了这些配置以后，各个软件的存在感反而不强了，因为几乎都会保持打开状态，交给配置文件去决定代理方式，大大提升了工作效率。

由于我没有使用安卓手机，所以暂时没有安卓平台的教程，读者可以自行搜索类似的代理软件并进行配置。

## VPS 配置

可以利用[这个脚本](https://github.com/nkwsqyyzx/.sys.config/blob/master/init.scripts/install-centos)快速配置 VPS 环境

给 VPS 开启 BBR 加速后，可以显著加快网络连接速度，我测试的效果是提高了 6 倍

如果使用 Digital Ocean，请参考[这个教程](http://w3cboy.com/post/2017/08/digital-ocean-bbr/)，其它厂商也类似

BBR 是谷歌研发的新的 TCP 拥塞控制算法，能够在有一定丢包率的网络链路上充分利用带宽，感兴趣的同学可以学习这篇文章：

[Linux Kernel 4.9 中的 BBR 算法与之前的 TCP 拥塞控制相比有什么优势？](https://www.zhihu.com/question/53559433)

