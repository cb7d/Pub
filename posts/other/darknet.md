---
title: "ObjC 的 weak 修饰"
date: 2018-06-21T10:46:42+08:00
showDate: true
draft: false
tags: ["blog","DarkNet"]
---

# Darknet

很多人对暗网都很好奇，各种电视剧和电影也或多或少的提到了这个网络，我觉得有必要专门写一篇博客来向大家分享一下

## What is Darknet

先看一下维基百科词条「暗网」的定义

> 暗网（英语：Dark web）是存在于黑暗网络、覆盖网络上的万维网内容，只能用特殊软件、特殊授权、或对计算机做特殊设置才能访问。暗网构成了深网的一小部分，深网网络没有被网络搜索引擎索引，有时“深网”这一术语被错误地用于指代暗网。 构成暗网的黑暗网络包括F2F的小型点对点网络以及由公共组织和个人运营的大型流行网络，如Tor、自由网、I2P和Riffle。暗网用户基于常规网络未加密的性质将其称为明网。Tor暗网可以称为洋葱区域（onionland），其使用网络顶级域后缀.onion和洋葱路由的流量匿名化技术。

要了解什么是暗网，首先要先了解网络世界都有哪几部分，

通俗来说，网络世界可以用这个列表来形容

- 明网
- 深网
    - 暗网

## Tor

Tor，全称为 the onion router。

可以说，正是洋葱路由促进了暗网世界的发展，

在1995年，由美国海军研究实验室的，开始研发了一哥项目，目的是在互联网上隐藏访问痕迹，由于其利用的技术像洋葱一样，剥开一层后仍会看到一层，所以叫做洋葱路由。

该技术最初由美国海军研究办公室和国防部高级研究项目署(DARPA)资助。早期的开发由Paul Syverson、 Michael Reed 和 David Goldschla领导。这三个人都是供职美国军方的数学家和计算机系统的研究人员。

Tor的最初目的并不是保护隐私，或者至少不是保护大部分人认为的那种“隐私”，它的目的是让情报人员的网上活动不被敌对国监控。在美国海军研究实验室1997 年的一篇论文中指出，“随着军事级别的通信设备日益依靠公共通讯网络，在使用公共通信基础设施时如何避免流量分析变得非常重要。此外，通信的匿名性也非常必要。”

该项目初期进展缓慢，到 2002 年，来自海军研究机构的Paul Syverson 还留在项目里，两个MIT的毕业生 Roger Dingledine 和 Nick Mathewson 加入了项目。 这两个人不是海军研究实验室的正式雇员。而是作为 DARPA和海军研究实验室的高可靠性计算系统的合同工方式加入的。在后来的几年里，这三个人开发了一个新版的洋葱路由，也就是后来的 Tor(The Onion Router)。

## Tor访问的原理

那么，Tor是如何工作的呢，

1. 访问发生时，客户端要先向目录服务器获取中继节点列表
2. 从节点列表中随机选择三个最优节点
3. 使用节点建立链路（circuit），每一步的访问都会通过TLS加密发送

那么你可能会问，这样就无法被追踪了吗？其实重点就在于建立链路发送数据包的过程

Tor的数据包Cell如下

| CircID | CMD   | DATA    |
| ------ | ----- | ------- |
| 2字节  | 1字节 | 509字节 |

其中，CircID由客户端随机生成，用来区分不同的链路，CMD表示数据包要执行的动作，DATA负载着发送的数据

在这里我们把客户端随机选择的三个中继服务器定义为 C1，C2，C3，


1. 客户端从服务列表获取三个中继节点服务器的公钥
2. 客户端生成一对密钥，CMD=create，DATA=C1的公钥加密客户端生成的公钥，构成数据包发送给C1
3. C1收到数据包检查CMD为建立链路，解开DATA，获取到客户端的公钥，同时生成一对临时密钥，CMD=created，DATA=客户端密钥加密C1生成的临时密钥中的公钥，发送给客户端
4. 客户端获取到C1生成的临时公钥
5. 客户端将要与C2建立连接，但是数据包要通过C1发送，CMD=relay，DATA=C1生成的临时公钥加密的C2信息
6. C1收到后解包，替代客户端与C2建立连接
7. 如上再继续重复C2与C3建立连接，整个链路就建立完成

隐秘踪迹的关键就在与看似C1知道客户端源IP，但是C1无从得知自己在链路的位置，从而无法判断客户端是否为中继服务器，所以无法追踪到信息的源头


## 使用 Tor 访问 Darknet
说了这么多，那么到底如何访问暗网呢

### Tor 浏览器

[Tor Browser 官方下载链接](https://www.torproject.org/download/)

下载对应的版本就好，我下载的是Mac版本，下载后安装

针对天朝的大内网，我们要进行一系列配置。

![dn1](https://github.com/FelixScat/Pub/blob/master/image/darknet/dn1.png?raw=true)

选择右边的configure

![dn2](https://github.com/FelixScat/Pub/blob/master/image/darknet/dn2.png?raw=true)

选择一个网桥

![dn3](https://github.com/FelixScat/Pub/blob/master/image/darknet/dn3.png?raw=true)

最后会创建一条链路

![dn4](https://github.com/FelixScat/Pub/blob/master/image/darknet/dn4.png?raw=true)

完成后，我们可以看到Tor浏览器是一个火狐浏览器的魔改版，使用DuckDuckGo作为默认搜索引擎，下面贴两张暗网交易市场的截图

![dn5](https://github.com/FelixScat/Pub/blob/master/image/darknet/dn5.png?raw=true)

![dn6](https://github.com/FelixScat/Pub/blob/master/image/darknet/dn6.png?raw=true)

![dn7](https://github.com/FelixScat/Pub/blob/master/image/darknet/dn7.png?raw=true)

![dn8](https://github.com/FelixScat/Pub/blob/master/image/darknet/dn8.png?raw=true)


### 常用的几个网址

这里给大家几个目前还在维护的暗网地址，有条件的童鞋可以前往观摩一下

- 搜索 not Evil http://hss3uro2hsxfogfq.onion/
- 交易市场 UnderMarket2.0 http://gdaqpaukrkqwjop6.onion/
- 暗网「淘宝」 交易市场（中文）http://deepmixaasic2p6vm6f4d4g52e4ve6t37ejtti4holhhkdsmq3jsf3id.onion/

## 注意事项！

可以看到如上，大部分运维在暗网的网站基本看起来像上个世纪的页面，也没什么很有趣的内容，所以在这里不建议大家频繁去访问，要保持一颗敬畏之心，不要相信网站所售卖的一些东西。

> 网络世界并非是不法之地。