---
title: "Go Goroutine的状态"
date: "2020-06-13 21:32:03"
categories:
  - Go
tags:
  - Go
toc: true
math: true
type: about
---

Goroutine status，一个Goroutine从生到死，一共有8种状态，2个GC状态。
<!-- more -->
### Gidle

表示该goroutine已经被分配，但是尚未初始化。\_Gidle = 0

### Grunnable

表示该goroutine正在排队（在等待运行队列），尚未执行用户代码，还没拥有stack。\_Grunnable=1

### Grunning

表示该goroutine正在运行，已经和M、P配对成功，拥有自己的stack。\_Grunning=2

### Gsyscall

表示该goroutine正在执行系统调用，不执行用户代码，拥有自己的堆栈，不在run queue。它被分配一个M。\_Gsyscall=3

### Gwaiting

表示该goroutine被阻塞，不执行用户代码，不在run queue。\_Gwaiting=4

### Gmoribund\_unused

这个本人尚不清楚用处。\_Gmoribund\_unused=5

### Gdead

表示该goroutine还没使用，可以刚刚退出、处于空闲列表或者刚刚被初始化。不执行用户代码，可能有stack。\_Gdead=6

### Genqueue\_unused

表示当前未使用。\_Genqueue\_unused=7

### Gcopystack

表示该goroutine的stack正在复制扩容。\_Gcopystack=8

### Gscan与Gscanning

表示GC正在扫描的堆栈。与Gscan不同，Gscanning是短暂的阻塞一下g，这对应了Go的GC的发展路线，从起初的STW到后面的引入混合写屏障的三色标记。而Gscan会让当前的goroutine停止。

\_Gscan         = 0x1000
\_Gscanrunnable = \_Gscan + \_Grunnable // 0x1001
\_Gscanrunning  = \_Gscan + \_Grunning  // 0x1002
\_Gscansyscall  = \_Gscan + \_Gsyscall  // 0x1003
\_Gscanwaiting  = \_Gscan + \_Gwaiting  // 0x1004