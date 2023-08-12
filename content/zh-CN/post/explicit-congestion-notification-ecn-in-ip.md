---
title: "Explicit Congestion Notification (ECN) in IP"
date: "2020-06-18 21:08:13"
categories:
  - TCP/IP
tags:
  - TCP/IP
toc: true
math: true
type: about
---

  TCP/IP的标记位

### IP的ECN(显式拥塞通知)

ECN使用IP首部的Tos字段最后2位(也称最低有效位)，根据[RFC 3168](https://tools.ietf.org/html/rfc3168#section-5)，可以得知这2位分别成为ECT和CE，ECT由发送方来设定是否支持ECN，而CE是由路由器来设定，通知接收方当前拥塞，这也意味着路由器要支持ECN。 ECT和CE有4种不同的组合00、01、10、11分别表示不同的状态。

*   00表示不支持ECN传输(Not-ECT)
*   01和10表示支持ECT传输——ECT(0)、ECT(1)
*   11表示发生拥塞(CE)。

### TCP的ECN

TCP头中新增三个flag来支持ECN，分别是Nonce Sum(NS)、ECN-Echo(ECE)和Congestion Window Reduced(CWR)。 NS用于防止TCP发送者的数据包标记被意外或恶意改动，而另外2位则是回传拥塞指示。当接收者收到IP报文，发现IP头部的ECN位为11，则会通知上层标记ECE位用于通知发送方减少发送速率。而发送方收到这个标记了ECE位的包后，发送速率减半，同时也会标记CWR位用于通知接收方已经确认阻塞指令。

 附一张图：

 ![](/images/explicit-congestion-notification-ecn-in-ip/20200628121725644.png)

### ECN与现有的TCP拥塞控制

现有的TCP拥塞控制都是在中间路由器丢包之后，TCP协议根据RTO超时来重传丢失的包，但是对于主机之间的TCP连接来说它不知道当前发送的链路正在拥塞，它要等待RTO计时器超时后开始重传报文和降低发送速率，这个RTO计时器超时过程花费个几秒或者十几秒等。(网络波动从轻载-过载-轻载) 而ECN机制可以让TCP发送端能够感知当前链路的拥塞程度，以此通过设置ECE标记来通知对方要降低发送速率(依靠路由器设置ECN位为11)，而不是静静的等待RTO超时再发送数据。

### 补充另外几个标记位

URG:一个正向偏移值，表示在包中的第几个字节开始有紧急数据要处理，直接交给上层协议，**不需要经过缓冲区** PSH:表示需要将收到的数据立即交给上次协议，但是不需要等到整个缓冲区填满，直接上交，**经过缓冲区**。