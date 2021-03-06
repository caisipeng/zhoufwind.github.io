---
layout: post
title: "为什么POST比GET更耗时？"
subtitle: 在进行网站开发时，遇到大并发场景下，若想提升服务QoS，请谨慎使用POST方法
date: 2018-06-28 00:40:00 +0800
header-img: "img/in-post/post-exp-than-get.png"
header-mask: 0.3
catalog: true
tags:
    - 运维
    - CDN
    - HTTP
---
### 背景
早上上班没多久，接到用户投诉，CDN平台显示当前5XX数量巨大，整站服务质量可能已经收到影响，我们这边必然是不敢怠慢，立即开始排查。

先看整站错误率，确实相对平时的几千个异常，今天上来就是上万肯定是有故障了。但并未接到监控报警和大量用户反馈，应该是某个站点的问题，立即对错误数进行排序，发现确实是某个域名返回了大量504错误造成的。

这是个已知问题，先前排查下来发现这个域名在各家CDN的响应状态码均出现5XX错误，而源站日志并未显示源站有任何异常，所以结论很简单：
> 源站没问题，错误都是CDN抛的，应该让CDN优化。

然而，真的是这样吗？

### 问题
针对这个问题，我们发现几个特征：
- 报错的都是POST请求
- 程序逻辑是每分钟进行一次POST请求
- 相同CDN节点响应其他GET请求均正常

类似频繁发起POST请求的站点在我们这里并不少，给我的感觉就是但凡发起POST、请求频率高、POST内容大的这些请求，很大可能都会在CDN这层发生异常，这是为什么呢？我想我们需要分几个方面来考虑

#### CDN原理方面
我们可以大致把CDN看做是用户与源站间的一个代理节点，CDN的本质是避免跨网跨距离传输造成的延迟，而在用户机房同城不跨网的情况下，多加一个代理节点反倒会增加请求耗时，再加上CDN内部还要一系列的回源导流，无疑是增加了请求的耗时。所以，从用户到CDN，CDN到源站这条链路上，任何一个节点的故障，都有可能导致用户请求失败，再加上每分钟一次的高频请求，更是对整条链路的网络质量提出了更高的要求。

#### HTTP协议方面
Yahoo在很早之前就发表过以下文章：

> POST is implemented in the browsers as a two-step process: sending the headers first, then sending data. So it's best to use GET, which only takes one TCP packet to send (unless you have a lot of cookies).[YSLOW]

多数浏览器对于POST采用两阶段发送数据的，先发送请求头，再发送请求体，即使参数再少再短，也会被分成两个步骤来发送（相对于GET），也就是第一步发送header数据，第二步再发送body部分。HTTP是应用层的协议，而在传输层有些情况TCP会出现两次连结的过程，HTTP协议本身不保存状态信息，一次请求一次响应。对于TCP而言，通信次数越多反而靠性越低，能在一次连结中传输完需要的消息是最可靠的，尽量使用GET请求来减少网络耗时。如果通信时间增加，这段时间客户端与服务器端一直保持连接状态，在服务器侧负载可能会增加，可靠性会下降。[南山磨刀]

### 缓解
取消web端的POST请求，同时优化CDN节点覆盖。

### 后记
一番讨论后，我们并无能力在短时间改变国内的网络质量，但是我们可以先从优化程序逻辑及改善CDN链路覆盖开始。

[YSLOW]: https://developer.yahoo.com/performance/rules.html
[南山磨刀]: https://segmentfault.com/q/1010000000213082