# 路由器爱国上网、屏蔽广告与宽带提速

在路由器中开启代理的好处很明显，最明显的就是连在这个路由器上的所有终端都不用考虑爱国上网的问题了。不管是 GMail 还是 Dropbox 这种服务都可以毫无顾忌的用起来了。

即使是在电脑的浏览器上有 [SwitchyOmega](https://chrome.google.com/webstore/detail/proxy-switchyomega/padekgcemlokbadohgkifijomclgjgif?utm_source=chrome-ntp-icon) 这样的神器，但终端和 APP 的爱国上网依然对很多人来说是个问题，比如经常出问题的 `npm instal` 和 Telegram、Dropbox 这类的桌面应用。

虽然很多路由器都自带 VPN，但恕我直言，VPN 是垃圾到无法使用的技术。因为它不支持透明代理，所谓的透明代理是指对于用户而言，无法察觉到自己被代理了。

以访问淘宝为例，如果是用 VPN 这种全局代理，就会被重定向到国际版的淘宝，导致速度大幅度降低。而如果是使用 shadowsocks 这类透明代理，会自动把淘宝、百度这类国内网站设置为不用代理，只有在访问谷歌等网站时才走代理。这个规则列表已经有人整理出来了，叫做 GFWList。

以上基本信息在我的上一篇文章：[全自动科学上网方案分享](./fq.md) 里面都有说过，就不多废话了。

文末会有宽带提速的教程，如果你还在用 20M 甚至 10M 的网络，那么每月只要十几块钱，就可以享受 100M 网络了，效果非常明显。

## 硬件选择

第一件事是要选择路由器，因为开启代理需要运行 shadowsocks 客户端，既然是软件，就一定要跑在操作系统上，自然会对硬件有一些要求。因此四五年前的硬件基本上就别想了。

这里我比较推荐 [小米路由 3G](http://union-click.jd.com/jdc?e=0&p=AyIHZRprEQISDlQbXCVGTV8LRGtMR1dGXgVFTUdGW0pADgpQTFtLH1sVCxMHUgQCUF5PNxBGHFd5elsuexhJRkYDVBsAfmNiBBMXV3sBEwdcB1oXHhEDRBtZHgIUDFESaxQyEgZUGl8TAhMAUCtrFQIiUTsbWhQDEwZQG1gXMhIEVhpZEAsVAlIrWxEBEg9RH1oTCxEDVitcJVlHaVMbXkcBFQNUGw9BBBQ3ZR9bFQsTB1IrayUyEjdW&t=W1dCFBBFC1pXUwkEAEAdQFkJBV8VAhsGVRxETEdOWg%3D%3D)，理由是价格便宜，两百出头一点，硬件性能足够（主要就是看 CPU、内存和闪存大小），两个 LAN 口和一个 WAN 口都是千兆的，在可预见的未来不会拖累网速。

# 操作流程

整个流程可以分为以下几步：

1. 刷开发版系统
2. 开启 SSH
3. 刷入 breed 启动引导
4. 刷入 Padavan 固件
5. 配置透明代理和去广告
6. 宽带提速

## 开启 SSH

前文说了，路由器就是一个小型的 Linux 操作系统，第一步自然就是要打开 SSH，这样才能在命令行里操作路由器。

> **注意下文所有固件均针对小米路由器 3G，请勿乱用，变成砖概不负责**

首先进入[小米路由器官网](http://www.miwifi.com/miwifi_download.html)，在 ROM 列表里找到找到 **ROM for R3G 开发版**，下载下来后拷贝到 FAT/FAT32 格式的 U盘中并重命名为 **miwifi.bin**

接下来先拔掉路由器电源线，再插入 U盘，然后找一根牙签或掏耳勺的柄戳住 Reset 键（在 U盘旁边的小洞里）。

最后接通电源，过大概 15 秒后路由器指示灯会变成黄灯并且一闪一闪的，此时松开 reset 键，等几分钟路由器就会重装好。

一般来说这一步没有风险。

## 开启 SSH

打开 <https://d.miwifi.com/rom/ssh>，下载工具包，放入 U盘中并重命名为 **miwifi_ssh.bin**。这个页面还会告诉你 Root 用户的密码，待会儿要用到。

然后重复之前的安装步骤（断电、插 U盘、按住 Rest、通电等黄灯闪烁、松开 Reset 等安装完成）。

路由器重启后，在终端输入 `ssh root@192.168.31.31`，然后输入之前记下的 Root 密码应该就连上了。

这一步基本也没有风险。

## 刷入 Breed 启动引导

理论上来说这一步是可选的，我们可以直接刷 Padavan 固件，但存在刷坏的风险。Breed 是社区开发的引导工具，用于替换系统的引导工具。

这样做的好处相当于是增加了一个中间层，只要刷入成功，后续就是在 Breed 里面搞事情了，不会影响系统，自然就不会变砖了。不管是更新固件还是切换回小米原装的固件，都非常容易。

这一步很容易，没有硬件操作，只要在连上 SSH 后输入以下命令就行：

```shell
cd /tmp
wget https://breed.hackpascal.net/breed-mt7621-xiaomi-r3g.bin
mtd -r write breed-mt7621-xiaomi-r3g.bin Bootloader
```

耐心等待路由器重启即可。

这一步风险较高，如果操作失误可能会导致路由器变砖。好在步骤比较简单，只要不作死，复制上面的命令来执行就不会出问题。

## 刷入 Padavan 控件

首先下载控件：<https://pan.baidu.com/s/1cweGGY>，如果地址失效就自行搜索，注意文件名是：**MI-R3G_3.4.3.9-099.trx**。

接下来我们需要进入刷机模式：断开电源，拔下 U盘，按住 Reset 键，插回电源，过几秒后指示灯开始闪烁，松开 Reset 键即可。

这一步和之前刷 ROM 和开启 SSH 的步骤类似，但注意不要插入 U盘。

然后在浏览器访问 192.168.1.1 就可以进入 Breed 的控制台。如果家里的路由器连在上级光猫路由器上，192.168.1.1 可能会被占用，此时拔掉小米路由器的 WAN 口网线即可。

最后在 Breed Web 控制台依次选择：固件更新 -> 常规固件 -> 勾选固件复选框 -> 浏览，选择刚刚下载好的 Padavan 固件上传，刷入搞定！

## 配置代理和去广告

重启之后通过 192.168.123.1 进入 Padavan 控制台，在左侧拓展功能中找到 Shadowsocks，填写 VPS 的信息，如果没有的话可以去 [搬瓦工](https://bwh1.net/aff.php?aff=19860&pid=55) 购买年费 39 刀，每月 2000G 中国直连流量的套餐。工作模式建议选择 GFWList 并自动更新，这样可以获取最新的爱国上网地址列表，实现透明代理。

同样是在左侧，找到 **广告屏蔽功能**，比较推荐用 koolproxy 来过滤广告。注意如果广告使用 HTTPS 是无法屏蔽的，因为插件无法判断某个请求是不是广告，这时候你有两种选择：

1. 按照 koolproxy 提供的根证书，存在一定的安全风险，但可以彻底屏蔽广告
2. 不用根证书，保证绝对安全，忍受 HTTPS 广告。

这里笔者不做推荐，用户自行评价

## 宽带提速

在左侧拓展功能->配置拓展环境中，打开迅雷快鸟，输入开通了迅雷快鸟的迅雷账号和密码即可。

笔者使用的是 50M 的北京联通宽带，实测开启后网速接近 100M，原理是迅雷与网络服务商达成了协议，授权迅雷接触网络运营商的网速限制。

由于不是所有用户通用，建议先自行测试下提速比例，合适再买。

## TODO

其实还有些小遗憾，主要是小米路由器 3G 自带的 USB3.0 接口没有被我有效利用起来，后续还可以添加远程迅雷下载和硬盘 FTP 共享的功能。