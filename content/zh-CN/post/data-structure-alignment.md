---
title: "go 内存对齐"
date: "2020-07-18 03:16:28"
categories:
  - Go
tags:
  - Go
toc: true
math: true
type: about
---

### 问题

在下面2个结构体所占的字节数又为多少？

```go
type A struct{
   a int8
   d int16
   c int32
   b int64
}

type B struct{
   a int8
   c int32
   b int64
   d int16
}
```



#### 答案

![](/images/data-structure-alignment/20200717180044881.png) 答案显而易见，如果仔细观察就会发现A、B两个结构体中的字段类型排序不同。这就涉及到了Data Structure Alignment(也就是内存对齐)的知识点了。

#### 32位与64位的区别

在了解什么是内存对齐前，首先要知道CPU、内存、地址总线、数据总线的概念。当cpu想要从内存中获取数据，就要通过地址总线传送地址给内存，内存在根据传送过来的地址得到数据，最后将数据从数据总线传给cpu。通常8根数据总线就寻址255byte的内存空间，而平常我们所说的32位和64位实际上是代表32根/64根数据总线，它们所对应可寻址空间位2^32和2^64。也就是说64位的机子当对于32位的机子的寻址空间更大了。

### 什么是Data Structure Alignment

#### 内存条的bank，chip，rank

(这里偏理解的东西，个人感觉不好口述，还是贴一下大佬的图（会在下面Refrence注明）) ![](/images/data-structure-alignment/20200717182436110.png) 

而8个Bank所对应的同一个位置的字节(8个位置)连起来就称为连续8个字节(这里标注为a0到a8)。但是如果不是从a0开始，而是从a1到a9呢？这时候就不能在一次操作中一次取出，而是通过二次操作，第一次取a1到a8，第二次取a9。这样就浪费了一次cpu的操作，而且更极端的情况就是一个占2字节的值分别在a8，a9上，这样不仅消费cpu，更浪费内存(因此明明可用a0、a1来解决同时又可以在一次操作完全取出)。所以编译器就会给不同类型的数据安排适合的内存地址，以此来提高性能。简单来说就是争取在cpu一次操作(操作8个字节)中，一次性读取出想要的数据。 因此Data Structure Alignment(内存对齐)出现便是作用于此。注意不同平台每种类型的对齐边界都会有所不同。

### 题目图解(64位window下)

![](/images/data-structure-alignment/20200717190554660.png)

### Reference

*   [Data Structure Alignment in Wikipedia](https://en.wikipedia.org/wiki/Data_structure_alignment)
*   [Is it necessary to use Data structure alignment](https://stackoverflow.com/questions/42053791/is-it-necessary-to-use-data-structure-alignment)
    
*   [Bilibili](https://www.bilibili.com/video/BV1Ja4y1i7AF?t=150)