---
title: "Thread-Caching Malloc"
date: "2020-08-02 11:40:59"
categories:
  - memory
toc: true
math: true
type: about
---

### 前言

关于TCMalloc的相关资料和讲解，网上有很多关于其细节文章，在这就不一一阐述。主要目的是大体的结合官方文档和图片来阐述，但是具体细节个人是希望通过查阅资料去理解，最重要是靠个人理解。

### 图解

虽然图是在网上借鉴的，但个人查阅相关资料，并图上加了一点自己的注释。 ![](/images/聊聊thread-caching-malloc/20200722022116565.png)

#### Central Free Lists 、Thread Cache Free Lists

关于Central Free Lists 、Thread Cache Free Lists这两个list分别对应着CentralCache和ThreadCache。CentralCache的主要工作是管理分配回收ThreadCache中的span。而ThreadCache的创建与销毁都是随着线程的创建与销毁进行的。 ![](/images/聊聊thread-caching-malloc/20200722022316165.png) 

初始化的主要工作是对PageHeap、CentralCache初始化、分配器和Size Class的初始化。当有线程需要申请内存时，ThreadCache会分配内存给它，如果不够就去CentralCache取。

#### 内存回收

关于内存回收这方面，建议参考文档和网上的一些资料，这方面偏理解。

#### 内存碎片

可以参考这片文章，注意里面的关键词语“对齐”--也就是所谓的内存对齐(可以参考我这篇博客 [Data Structure Alignment](https://202.182.125.62/wp-admin/post.php?post=302&action=edit))，链接：[点击](https://wallenwang.com/2018/11/tcmalloc/#ftoc-heading-45)

### 思考：TCmalloc的好处

在上面的第一张图，可以知道TCmalloc给小对象做了三层的缓存服务，如果不够就会一步一步的向后面的服务获取内存，而向PageHeap、CentralCache内的内存都是提前向系统申请好的。因此线程不需要直接通过系统调用申请，而是申请TCmalloc所分配好的内存，从而使得整个过程都处于一种用户态的形式，不需要进入内核态。比如C函数库中的内存分配函数malloc()就涉及到一次用户态到内核态之间的切换。通过以这种形式来提供性能。

### 应用

Go的内存模型也是基于TCmalloc的设计思想。不过我在[百度百科](https://baike.baidu.com/item/tcmalloc/2832982?fr=aladdin)上面还发现了TCmalloc还可以应用在MySQL上，让它在高并发下内存占用更加稳定，这也是对mysql的一种不错的优化。

### Reference

*   [https://gperftools.github.io/gperftools/tcmalloc.html](https://gperftools.github.io/gperftools/tcmalloc.html)
*   [https://github.com/google/tcmalloc](https://github.com/google/tcmalloc)
*   [https://github.com/google/tcmalloc/blob/master/docs/design.md](https://github.com/google/tcmalloc/blob/master/docs/design.md)
*   [https://wallenwang.com/2018/11/tcmalloc/#ftoc-heading-36](https://wallenwang.com/2018/11/tcmalloc/#ftoc-heading-36)