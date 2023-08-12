---
title: "nginx epoll"
date: "2020-07-01 01:52:28"
categories:
  - nginx
tags:
  - nginx
toc: true
math: true
type: about
---

### 前言

  最近在调试个人博客时，发现服务器上的nginx居然没有配置epoll事件驱动，在之前学习Go的net包时，也接触过底层源码，也了解过其底层也是使用了epoll。为此我也专门去了解epoll，还有select
poll。

### Nginx配置epoll

在nginx.conf下配置事件驱动模型epoll

```
events {
   accept_mutex on; #设置网路连接序列化，防止惊群现象发生，默认为on
   multi_accept on; #设置一个进程是否同时接受多个网络连接，默认为off
   use epoll; #事件驱动模型，selectpollkqueueepollresig/dev/polleventport
   worker_connections 1024; #最大连接数，默认为512
}
```

特别要注意一点在events里的accept_mutex配置涉及到一个[Thundering herd problem](https://en.wikipedia.org/wiki/Thundering_herd_problem)(**惊群问题**)需要我们留意。 其他的一些配置就不再多说，配置也比较简单，主要还是走官网文档。记得修改过后要重启nginx

### 关于epoll

其实epoll、select、poll都是I/O多路复用的机制，本质上还是同步I/O需要自己负责进行读写。而select、poll都是出现在epoll之前，select的机制就是使用轮询的方式扫描fd，同时在其过程还涉及到内核态与用户态来回复制fd，效率低下。poll的机制原理与select十分相似，主要一点不同是poll采取fd的结构是链式，没有最大连接数限制，而select的最大连接数默认为1024。 关于epoll的一些组成在这就不必多说，网上都有，关键是要理解其内部的各组件的作用。

#### API

关于epoll，它有三个API分epoll_create、epoll_ctl、epoll_wait，原本对于这三个API我还有不太理解除了epoll_create。于是Google了一下，发现里面大有文章。 在epoll_create会在内核产生一个epoll的fd，而之后的epoll_ctl、epoll_wait都是以它为中心进行操作，epoll_ctl可以理解为一个事件监听器，它可以把需要监听的事件全都注册进epoll的fd中，并用红黑树进行存储。如果事件触发，那么epoll会从这个事件插入到一个链表中，最后通过epoll_wait进行调用。

#### 源码

关于源码部分，我在Github上找到一个epoll的源码分析，里面分析的十分详细，不仅包括了完整源码，而且还分析了epoll是如何得知fd的状态变化的，具体深入到内核的poll技术，可以去参考一下[这里](https://github.com/Liu-YT/IO-Multiplexing/blob/master/%E6%BA%90%E7%A0%81%E5%89%96%E6%9E%90/epoll.md)。