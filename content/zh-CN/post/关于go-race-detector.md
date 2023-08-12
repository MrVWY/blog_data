---
title: "Go Race Detector"
date: "2020-07-16 12:30:51"
categories:
  - Go
tags:
  - Go
toc: true
math: true
type: about
---

碰巧在看Go question的PDF时，意外发现在其上面所推荐的博客上看到一篇关于[map 并发崩溃一例](https://xargin.com/map-concurrent-throw/)的文章，他所使用的检查bug方法引起了我的注意和回想。
<!-- more -->

### 前言

在那篇文章中，他所使用检测bug命令是go run -race ×.go。当时-race就引起了我的注意，因此在前几篇关于Go的文章中，在阅读源码的过程中我经常看到这个关于race包的引用，当时我是不知道这里这一小段的代码具体作用是干啥的，直到今晚看到这篇关于[map 并发崩溃一例](https://xargin.com/map-concurrent-throw/)的文章我才知道其的实际作用。 下面随手贴几段关于race包在各处的代码

```go
//chan.go
func send(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
   if raceenabled {
      if c.dataqsiz == 0 {
         racesync(c, sg)
      } else {
         qp := chanbuf(c, c.recvx)
         raceacquire(qp)
         racerelease(qp)
         raceacquireg(sg.g, qp)
         racereleaseg(sg.g, qp)
         c.recvx++
         .......
      }
   }
.......
}

func closechan(c *hchan) {
   .......
   if raceenabled {
      callerpc := getcallerpc()
      racewritepc(c.raceaddr(), callerpc, funcPC(closechan))
      racerelease(c.raceaddr())
   }
   .........
}
//rwmutex.go
func (rw *RWMutex) RLock() {
   if race.Enabled {
      _ = rw.w.state
      race.Disable()
   }
   .....
   if race.Enabled {
      race.Enable()
      race.Acquire(unsafe.Pointer(&rw.readerSem))
   }
}
```

可以看出race包在Go源码中是十分常见的。

### race是什么

根据官网文档，-race选项用于检测数据竞争。在开启了race检测之后，如果程序发生了例如map的竞争，它会一层一层的将错误打印出来。常用于并发程序的检测。具体的操作和说明在这就不多说，请移步到下面的参考文献查阅。

### 使用方法

race检测已经添加到go tool chan中，可以在命令行中加入-race，即可开启。 类如：go run -race 、go test -race等 参考文献及案例

*   [The Go Blog-Introducing the Go Race Detector](https://blog.golang.org/race-detector)
*   [Data Race Detector](https://golang.org/doc/articles/race_detector.html)