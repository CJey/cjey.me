---
title: 记一次网络诊断 - 切换wifi后主机无法访问服务器
date: 2015-08-23 12:57:27
tags:
  - 有趣
  - 疑难杂症
  - 恍然大悟
categories:
  - 网络
  - 网络层
---

# 背景

公司研发的硬件, 运行自行定制的Android ROM(基于AOSP 4.4), 偶尔会发生一个怪异的网络问题

* 主机wifi首先接入当前网络A, 一切没问题(可以访问IDC服务器); 切换接入到另一个网络B, 无法访问IDC服务器, 但此时在网络B的主机访问其他的Internet服务并无问题
* 主机wifi首先接入网络B, 一切没问题; 切换接入到一个网络A, 一切没问题; 再切回到网络B, 无法访问IDC服务器, 访问其他Internet服务并无问题

这个问题在办公室环境下发生频率较高(但并非100%), 也有部分客户会抱怨切换网络后不能访问服务器

公司网络A的结构不同于一般家用网络, 默认网关和对外的出口路由器并不是同一台设备, 结构示意如下

>DHCP
>Network: 192.168.0.0/24
>Gateway: 192.168.0.1, 负责公司内部各网络间的数据交换
>DNS: 192.168.0.201
>Router: 192.168.0.254, 负责提供Internet接入

先说原因/结论: **主机的ROM不能正确释放由ICMP Redirect报文控制生成的临时路由**

下文都是一些分析流水, 如果你有些网络经验的话, 应该可以不用继续往下看了

<!-- more -->

# 初次排查

刚发生这个问题的时候, 大家都很困惑, 因为除了主机外, 其他的设备全部没有这个问题(手机, 平板, PC), 自然会认为问题是出在主机上, 不过因为在客户手上的机器并没怎么出现过这个问题, 所以, 公司内部测试的时候会避免这个问题

随着业务规模扩大, 主机遭遇的网络环境各种各样, 有更多的客户反馈这个问题, 于是在一个周末, 几个同事一起分析这个问题

1. 起初, 我们怀疑是DNS被劫持, 但在不同网络下ping IDC服务器可以看得到dns解析没问题
1. 奇怪的是, 在A网络下直接ping IDC的IP可以ping通, 切换到B网络下却一直提示unreachable
1. 而同样的, 在A网络下ping baidu.com可以ping通, 切换到B网络下也仍能ping通baidu.com, 仔细观察了一下发现, baidu.com做了dns balance, 所以测试中先后两次其实ping的是不同的IP
1. A网络下选定一个baidu的IP直接ping可以ping通, 切换到B网络下也仍能ping通, 这下就很头疼了, 因为就IDC的IP无法ping通
1. 重复了一次4, 这次发现baidu的IP和IDC的IP一个表现, B网络下也不能ping通
1. 翻看Linux的路由表, 并没有什么特别
1. 大家都陷入了沉思...
1. 重复测试了几遍, 结果和第一次没什么差别, IDC的IP表现一直一致, baidu的IP一会正常一会儿不正常
1. 引入新的网络C, 在网络B和网络C之间反复切换多次对比测试, 一切都正常
1. 有十几年经验的IT经理做了一个较合理的猜想: Linux自己控制着一张隐藏的路由表, 主机因为某个bug, 在切换网络时, 隐藏的路由表没有同步更新, 导致同样的IP在不同的网络下不能正确寻路
1. 最后的结论是: 公司的网络架构较一般环境复杂, 通常情况下不会有客户会处在这么复杂的架构下, 如果发生了这种情况, 尽可能帮助客户简化网络复杂度

虽然这次分析并没有找到真正的原因, 但是IT经理的猜想对我产生挺大冲击的

因为在我看来, **操作系统该做的是维护和管理好资源, 对用户提供抽象的高级管理接口**

就路由而言, 理当遵照我们所看到的路由表那样去工作, 怎么会偷偷搞一张用户无法控制的隐藏路由表呢?

可是, 只有这个解释最符合问题表现和大家的认知, 但是我也因此而困惑

不过后来的某一天, 在一篇讲述Linux下策略路由的文章中了解到, Linux下有255张路由表, 普通的路由指令显示的只是其中一张较低优先级的默认路由表而已, 突然, 我几乎就信了, 只要查出出问题的那张路由表不就可以解决这个网络问题吗!

这时, 我突然特别佩服IT经理, 能大胆的做出这么一个事实正确的猜想

不过, 进展仍然止步于此, 因为之后我们没有再去讨论这个问题, 我也没再去分析出问题的主机网络状况

# 问题再现

最近仍有客户反馈这个问题, 在上周, 硬件部和IT部一起重新搭建一个网络环境, 用来调查这个问题, 因为硬件部担心问题可能是出在wifi芯片上

当时IT的同事找到正在码代码的我(我并不清楚他们在做什么测试), 向我咨询一个奇怪的问题, 描述如下

主机接入新网络X, 一切正常, 切换到公司网络A, 一切正常, 切换回网络X, 不能ping通网络X中的网关
而此时, 接入网络X的手机, 并没有异常, 可以ping通网关, 诡异的是, 居然也能ping通主机

一开始听到这个描述, 我直接摇头否认这个问题的描述, 因为, 我认为, 同在一个简单链路内, 没有理由不能相互ping通(没有防火墙), 更何况问题发生时, 主机和手机之间是可以相互ping通的, 而wifi环境下所以数据包都需要经过AP中转, 而AP同时也是路由器, 没理由数据包能被路由转发到其他设备, 但路由自己却收不到

可是, 当我坐在主机前, 做了一些验证后, 我惊呆了, 居然真的不能ping通网关, 要知道, 在同一个链路内, 数据之间的传递可以完全只依赖自己的网卡就能完成

# 最终分析

查看路由表, 切换网络前后的路由表都看不出异常, 这时我突然想起来查一下"隐藏的路由表", 可是最后翻遍所有的路由表, 发现没有任何异常, 看来上一次的分析结论和Linux的255张路由表没有关系

没有什么头绪, 抓包吧, 一边ping网关, 一边tcpdump抓包, 终于! 露出马脚了!

网络X的网关是192.168.1.1, 网络A的网关是192.168.0.1, 网络A的出口路由是192.168.0.204

当我接入在网络X下, ping着网络X的网关时, tpcudmp抓包却看到系统一直在广播ARP查找192.168.0.204这个IP, 明明切换了网络, 192.168.0.204根本就不再出现网络X上, 而这个IP又正是网络A的出口路由(非网关), 这一定不是巧合!

重启主机, 从没接入任何网络时就开始就抓包, 抓住整个测试过程中的所有包, 看能否找出原因

1. 在初次接入网络X的情况下, 没有发现异常
1. 切换到网络A下, 果然, 频繁收到ICMP的重定向包, 指示相关的IP访问可以由192.168.0.204转发完成, 注意, 这里的重定向不仅仅只有公司的API服务器IP, 还有网络X的网关192.168.1.1(由于接入过网络X, 某些应用可能抓取了X下的网关做网络连通性测试用途)
1. 切换回网络X, 重新ping网关, 真的ping不通了, 而且抓到的全部都是探寻IP地址192.168.0.204的ARP数据包

不过, 找来更多的机器试图重现这个问题, 却发现这问题并不能完全重现, 继续分析了一下, 原因在于路由器发送ICMP重定向包的存在一个不污染网络的策略, 存在一个频率限制, 外部看起来呈现一定的随机性, 针对这个小问题, 在网络A的网关上人为伪造ICMP重定向包每3秒发送一次, 最终做到了所有机器全部可以重现问题

# 解决方案

很简单, 调整内核参数, 忽略所有ICMP重定向包

```
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.accept_redirects = 0
```

PS: 尽管问题出现在ROM本身, 但是最终没能查到原因