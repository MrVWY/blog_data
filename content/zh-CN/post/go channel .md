---
title: "Channel in Go"
date: "2020-06-13 12:08:50"
categories:
  - Go
tags:
  - Go
toc: true
math: true
type: about
---

### chan结构

```go
type hchan struct {
   qcount   uint           // buffer的总数据量
   dataqsiz uint           // circular(圆) queue的大小（可以理解为环形链表）
   buf      unsafe.Pointer // 一个指向buffer某个数据的指针
   elemsize uint16
   closed   uint32
   elemtype *_type // element type
   sendx    uint   // 已发送的索引位置的下标
   recvx    uint   // 已接受的索引位置的下标
   recvq    waitq  // 接收队列
   sendq    waitq  // 发送队列
   lock mutex
}


type waitq struct {
   first *sudog //sudog表示一个在等待队列的g
   last  *sudog
}
```




### makechan

makechan里面主要是对hchan、buffer等一些状态检测和调用mallocgc函数分配内存等。不过值得主要的地方是最后return的是一个\*hchan。因此在平常使用channel是依据指针操作的，而不是值的复制。

### chan send

#### 初始

```go
func chansend1(c *hchan, elem unsafe.Pointer) {
   chansend(c, elem, **true**, getcallerpc())
}
```

这里主要得知block字段都是以true值向下转递。

```go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {}
```

c表示某个chan，ep表示发送数据的地址，block表示是否为阻塞，callerpc表示调用地址，通过getcallerpc函数获取，返回的是一个程序的PC地址。同时其还有一个孪生函数getcallersp，返回的是一个函数的SP地址。

#### 判断状态

1、如果c为nil，是非阻塞状态直接返回false。否则通过gopark函数去永远关闭goroutine。这里提醒一句在golang中基本的channel的读写都是阻塞的。如果要非阻塞，可以在select中加入defalut语句。

```go
if c == nil {
   if !block {
      return false
   }
   gopark(nil, nil, waitReasonChanSendNilChan, traceEvGoStop, 2)
   throw("unreachable")
}
```

2、这里有点绕。个人见解为，当c不为nil和非阻塞时，要判断当前channel是否为关闭状态，因此channel可以在一瞬间就关闭了。同时如果buffer中没有数据和没有接收者就返回false。

```go
if !block && c.closed == 0 && ((c.dataqsiz == 0 && c.recvq.first == nil) 
   (c.dataqsiz > 0 && c.qcount == c.dataqsiz)) {
   return false
}
```



#### 加锁、添加数据

3、加锁，判断channel是否关闭。这里结合第2步可得知，c.closed = 0 表示该channel没有被关闭，而c.closed != 0 表示channel已经被关闭。

```go
lock(&c.lock)

if c.closed != 0 {
   unlock(&c.lock)
   panic(plainError("send on closed channel"))
}
```

4、从接收队列中取出一个接收者通过send函数发送数据

```go
if sg := c.recvq.dequeue(); sg != nil {
   send(c, sg, ep, func() { unlock(&c.lock) }, 3)
   return true
}

func send(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
   .........
   if sg.elem != nil {
      sendDirect(c.elemtype, sg, ep)
      sg.elem = nil
   }
   gp := sg.g
   unlockf()
   gp.param = unsafe.Pointer(sg)
   if sg.releasetime != 0 {
      sg.releasetime = cputicks()
   }
   goready(gp, skip+1)
}
```

先说下send函数的参数，c表示某个channel，sg表示某个goroutine接收者，ep表示要发送的数据，unlockf表示一个解锁channel的函数，skip字面意思是跳跃，具体不清楚，个人感觉是跟内存偏移量之类的相关。 具体流程大致为，先判断sg.elem是否为nil，如果不为nil的话，就sendDirect（重定向）到sg的指定位置，这里很好理解，因此sg代表的是一个goroutine，其在堆栈里面必然含有一定的数据，因此要重定向到其堆栈的空内存块存储。之后就通过goready函数将其唤醒，加入到go的调度中。 

5、向缓冲区添加数据

```go
if c.qcount < c.dataqsiz {
   qp := chanbuf(c, c.sendx)
   if raceenabled {
      raceacquire(qp)
      racerelease(qp)
   }
   typedmemmove(c.elemtype, qp, ep)
   c.sendx++
   if c.sendx == c.dataqsiz {
      c.sendx = 0
   }
   c.qcount++
   unlock(&c.lock)
   return true
}
```

6、当缓冲区满时 对于非阻塞channel，直接返回false

```go
if !block {
   unlock(&c.lock)
   return false
}

对于阻塞channel，挂起goroutine并将其抽象成sudog放进发送队列中

gp := getg()
mysg := acquireSudog()
mysg.releasetime = 0
if t0 != 0 {
   mysg.releasetime = -1
}
mysg.elem = ep
mysg.waitlink = nil
mysg.g = gp
mysg.isSelect = false
mysg.c = c
gp.waiting = mysg
gp.param = nil
**c.sendq.enqueue(mysg)**
goparkunlock(&c.lock, waitReasonChanSend, traceEvGoBlockSend, 3)
```



#### 标记可达

7、通过keepAlive函数把发送数据ep标记为可达，防止gc。同时清理工作现场。

```go
KeepAlive(ep)

// someone woke us up.
if mysg != gp.waiting {
   throw("G waiting list is corrupted")
}
gp.waiting = nil
if gp.param == nil {
   if c.closed == 0 {
      throw("chansend: spurious wakeup")
   }
   panic(plainError("send on closed channel"))
}
gp.param = nil
if mysg.releasetime > 0 {
   blockevent(mysg.releasetime-t0, 2)
}
mysg.c = nil
releaseSudog(mysg)
return true
```



### chan recv

代码与chan send流程大致相同但具体细节方面不同，不再多述。

### chan close

主要是对reader和wirter进行关闭

```go
// release all readers
//glist就是存放g的列表
for {
   sg := c.recvq.dequeue()
   if sg == nil {
      break
   }
   if sg.elem != nil {
      typedmemclr(c.elemtype, sg.elem)//这里可能就是对channel还存放的数据进行操作
      sg.elem = nil
   }
   if sg.releasetime != 0 {
      sg.releasetime = cputicks()
   }
   gp := sg.g
   gp.param = nil
   glist.push(gp)
}

// release all writers (they will panic)
for {
   sg := c.sendq.dequeue()
   if sg == nil {
      break
   }
   sg.elem = nil
   if sg.releasetime != 0 {
      sg.releasetime = cputicks()
   }
   gp := sg.g
   gp.param = nil
   glist.push(gp)
}
```

 

### race0.go与race.go

在channel的代码上出现了很多调用race0.go里面的方法，个人百度了一下从官网上找到了相关的解释为这是一个go tool chain。详情请移步[这里](https://blog.golang.org/race-detector)。一般情况下由于raceenabled为false其不会执行里面的函数。

### 写屏障

在channel分别出现了2个不同的写屏障方法分别是typeBitsBulkBarrier()和bulkBarrierPreWrite()。 typeBitsBulkBarrier()：出现在recvDirect()和sendDirect()中，是对每个指针复制到另外一个指针执行写屏障(这方面理解不太清楚，正确理解请自行Google)。个人理解是当接受/发送数据时，当前goroutine正在运行，而GC已经把所用到的对象已经关联起来，如果这时不执行写屏障，那么在下一次GC中所接收/发送的数据就会被清除。 bulkBarrierPreWrite()：出现在closechan()中，对在一定范围的内存中的每个指针执行写屏障。个人理解为当在关闭channel时，如果在channel中还有数据，那么就把它复制到另一块内存中。这也是当关闭管道之后，还能读取数据。

### 总结

总的来说channel内部结构是一个环形链表和一个接收者/发送者队列所组成。当对其发送或者接收数据时，都触发写屏障将数据复制到对应的goroutine中或者将数据写进channel中。而当channel中的buffer满时或者没有数据给goroutine接收时，这时候会将g抽象成sudog类存放在revc/send队列中，这里面会涉及到goroutine的暂停和唤醒操作。