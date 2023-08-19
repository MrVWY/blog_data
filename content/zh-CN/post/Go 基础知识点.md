---
title: "Go 基础知识点"
date: "2021-09-12 15:49:10"
categories:
  - Go
tags:
  - Go
toc: true
math: true
type: about
---

**记录一下golang的基础要点，以方便后续复习**

### GC

#### STW(stop the world)
  为了避免在 GC 的过程中，对象之间的引用关系发生新的变更，使得GC的结果发生错误（如GC过程中新增了一个引用，但是由于未扫描到该引用导致将被引用的对象清除了），停止所有正在运行的协程。  

  STW对性能有一些影响，Golang目前已经可以做到1ms以下的STW

#### 标记清除

​	标记清除（Mark-Sweep）算法是最常见的垃圾收集算法，标记清除收集器是跟踪式垃圾收集器，其执行过程可以分成标记（Mark）和清除（Sweep）两个阶段：

1. 标记阶段 — 从根对象出发查找并标记堆中所有存活的对象；
2. 清除阶段 — 遍历堆中的全部对象，回收未被标记的垃圾对象并将回收的内存加入空闲链表；

![](/images/Go基础知识点/GC-1.png)

标记阶段结束后会进入清除阶段，在该阶段中收集器会依次遍历堆中的所有对象，释放其中没有被标记的 B、E 和 F 三个对象并将新的空闲内存空间以链表的结构串联起来，方便内存分配器的使用。

![](/images/Go基础知识点/GC-2.png)

​	标记清除算法：垃圾收集器从垃圾收集的根对象出发，递归遍历这些对象指向的子对象并将所有可达的对象标记成存活；标记阶段结束后，垃圾收集器会依次遍历堆中的对象并清除其中的垃圾，整个过程需要标记对象的存活状态，**用户程序在垃圾收集的过程中也不能执行**（STW），我们需要用到更复杂的机制来解决这个 STW （Stop The World）的问题。

#### 三色标记算法

​	为了解决原始标记清除算法带来的长时间 STW，多数现代的追踪式垃圾收集器都会实现三色标记算法的变种以缩短 STW 的时间。三色标记算法将程序中的对象分成白色、黑色和灰色三类[4](https://draveness.me/golang/docs/part3-runtime/ch07-memory/golang-garbage-collector/#fn:4)：

- 白色对象 — 潜在的垃圾，其内存可能会被垃圾收集器回收；
- 黑色对象 — 活跃的对象，包括不存在任何引用外部指针的对象以及从根对象可达的对象；
- 灰色对象 — 活跃的对象，因为存在指向白色对象的外部指针，垃圾收集器会扫描这些对象的子对象；

​	在垃圾收集器开始工作时，程序中不存在任何的黑色对象，垃圾收集的根对象会被标记成灰色，垃圾收集器只会从灰色对象集合中取出对象开始扫描，当灰色集合中不存在任何对象时，标记阶段就会结束。

![](/images/Go基础知识点/GC-3.png)

​	三色标记垃圾收集器的工作原理很简单，我们可以将其归纳成以下几个步骤：

1. 从灰色对象的集合中选择一个灰色对象并将其标记成黑色
2. 将黑色对象指向的所有对象都标记成灰色，保证该对象和被该对象引用的对象都不会被回收
3. 重复上述两个步骤直到对象图中不存在灰色对象
4. 通过写屏障(write-barrier)检测对象有变化，重复以上操作
5. 收集所有白色对象（垃圾）

​	当三色的标记清除的标记阶段结束之后，应用程序的堆中就不存在任何的灰色对象，只能有黑色的存活对象以及白色的垃圾对象，垃圾收集器可以回收这些白色的垃圾。

#### 屏障机制

​	因为用户程序可能在标记执行的过程中修改对象的指针，所以三色标记清除算法本身是不可以并发或者增量执行的，它仍然需要 STW。

​	本来不应该被回收的对象却被回收了，这在内存管理中是非常严重的错误，我们将这种错误称为悬挂指针，即指针没有指向特定类型的合法对象，影响了内存的安全性[5](https://draveness.me/golang/docs/part3-runtime/ch07-memory/golang-garbage-collector/#fn:5)，想要并发或者增量地标记对象还是需要使用屏障技术。

想要在并发或者增量的标记算法中保证正确性，我们需要达成以下两种三色不变性（Tri-color invariant）中的一种：

- 强三色不变性 — 黑色对象不会指向白色对象，只会指向灰色对象或者黑色对象；
- 弱三色不变性 — 黑色对象指向的白色对象必须包含一条从灰色对象经由多个白色对象的可达路径

![](/images/Go基础知识点/GC-4.png)

​	上图分别展示了遵循强三色不变性和弱三色不变性的堆内存，遵循上述两个不变性中的任意一个，我们都能保证垃圾收集算法的正确性，而屏障技术就是在并发或者增量标记过程中保证三色不变性的重要技术。

​	垃圾收集中的屏障技术更像是一个钩子方法，它是在用户程序读取对象、创建新对象以及更新对象指针时执行的一段代码，根据操作类型的不同，我们可以将它们分成读屏障（Read barrier）和写屏障（Write barrier）两种，因为读屏障需要在读操作中加入代码片段，对用户程序的性能影响很大，所以编程语言往往都会采用写屏障保证三色不变性。

​	造成引用对象丢失的条件：

- 一个黑色的节点A新增了指向白色节点C的引用
- 并且白色节点C没有除了A之外的其他灰色节点的引用，或者存在但是在GC过程中被删除了

​	因此要满足弱三色不变性和强三色不变性，即可保证对象不丢失

1. Dijistra 插入写屏障：满足强三色不变性：黑色节点不允许引用白色节点，当黑色节点新增了白色节点的引用时，将对应的白色节点改为灰色。

   标记过程：

   - 垃圾收集器将根对象指向 A 对象标记成黑色并将 A 对象指向的对象 B 标记成灰色；
   - 用户程序修改 A 对象的指针，将原本指向 B 对象的指针指向 C 对象，这时触发写屏障将 C 对象标记成灰色；
   - 垃圾收集器依次遍历程序中的其他灰色对象，将它们分别标记成黑色；

   ![](/images/Go基础知识点/GC-5.png)
   

   缺点：

   ​		Dijkstra 的插入写屏障是一种相对保守的屏障技术，它会将**有存活可能的对象都标记成灰色**以满足强三色不变性。在如上所示的垃圾收集过程中，实际上不再存活的 B 对象最后没有被回收；而如果我们在第二和第三步之间将指向 C 对象的指针改回指向 B，垃圾收集器仍然认为 C 对象是存活的，这些被错误标记的垃圾对象只有在下一个循环才会被回收。

   ​		插入式的 Dijkstra 写屏障虽然实现非常简单并且也能保证强三色不变性，但是它也有明显的缺点。因为栈上的对象在垃圾收集中也会被认为是根对象，所以为了保证内存的安全，Dijkstra 必须为栈上的对象增加写屏障或者在标记阶段完成重新对栈上的对象进行扫描，这两种方法各有各的缺点，前者会大幅度增加写入指针的额外开销，后者重新扫描栈对象时需要暂停程序

2. Yuasa 删除写屏障：满足弱三色不变性：黑色节点允许引用白色节点，但是该白色节点有其他灰色节点间接的引用（确保不会被遗漏）当白色节点被删除了一个引用时，悲观地认为它一定会被一个黑色节点新增引用，所以将它置为灰色

#### 混合写屏障

​	过程：

1. GC开始时将栈上全部对象标记为黑色（之后不再进行第二次重复扫描，无需STW）

2. GC期间，任何在栈上创建的对象，均为黑色。

3. 被删的对象标记为灰色。

4. 被添加的对象标记为灰色。

#### GC 的触发条件

1. 主动触发(手动触发)，通过调用 runtime.GC 来触发 GC，此调用阻塞式地等待当前 GC 运行完毕。

2. 被动触发，分为两种方式：

   - 使用系统监控，当超过两分钟没有产生任何 GC 时，强制触发 GC。

   - 使用步调（Pacing）算法，其核心思想是控制内存增长的比例,每次内存分配时检查当前内存分配量是否已达到阈值（环境变量 GOGC）：默认 100%，即当内存扩大一倍时启用 GC。

### GPM 调度 和 CSP 模型

#### CSP 模型

CSP 模型是“以通信的方式来共享内存”，不同于传统的多线程通过共享内存来通信。用于描述两个独立的并发实体通过共享的通讯 channel (管道)进行通信的并发模型。

#### GPM 

##### G（Goroutine）

​		即 Go 协程（Goroutine），每个 go 关键字都会创建一个协程。Goroutine 在 Go 语言运行时使用私有结构体 [`runtime.g`](https://github.com/golang/go/blob/41d8e61a6b9d8f9db912626eb2bbc535e929fefc/src/runtime/runtime2.go#L404) 表示，结构体 [`runtime.g`](https://github.com/golang/go/blob/41d8e61a6b9d8f9db912626eb2bbc535e929fefc/src/runtime/runtime2.go#L404) 的 `atomicstatus` 字段存储了当前 Goroutine 的状态。除了几个已经不被使用的以及与 GC 相关的状态之外，Goroutine 可能处于以下 9 种状态：

| 状态          | 描述                                                         |
| ------------- | :----------------------------------------------------------- |
| `_Gidle`      | 刚刚被分配并且还没有被初始化                                 |
| `_Grunnable`  | 没有执行代码，没有栈的所有权，存储在运行队列中               |
| `_Grunning`   | 可以执行代码，拥有栈的所有权，被赋予了内核线程 M 和处理器 P  |
| `_Gsyscall`   | 正在执行系统调用，拥有栈的所有权，没有执行用户代码，被赋予了内核线程 M 但是不在运行队列上 |
| `_Gwaiting`   | 由于运行时而被阻塞，没有执行用户代码并且不在运行队列上，但是可能存在于 Channel 的等待队列上 |
| `_Gdead`      | 没有被使用，没有执行代码，可能有分配的栈                     |
| `_Gcopystack` | 栈正在被拷贝，没有执行代码，不在运行队列上                   |
| `_Gpreempted` | 由于抢占而被阻塞，没有执行用户代码并且不在运行队列上，等待唤醒 |
| `_Gscan`      | GC 正在扫描栈空间，没有执行代码，可以与其他状态同时存在      |

上述状态中比较常见是 `_Grunnable`、`_Grunning`、`_Gsyscall`、`_Gwaiting` 和 `_Gpreempted` 五个状态

![](/images/Go基础知识点/GMP-1.png)


​	虽然 Goroutine 在运行时中定义的状态非常多而且复杂，但是可以将这些不同的状态聚合成三种：等待中、可运行、运行中，运行期间会在这三种状态来回切换：

- 等待中：Goroutine 正在等待某些条件满足，例如：系统调用结束等，包括 `_Gwaiting`、`_Gsyscall` 和 `_Gpreempted` 几个状态；
- 可运行：Goroutine 已经准备就绪，可以在线程运行，如果当前程序中有非常多的 Goroutine，每个 Goroutine 就可能会等待更多的时间，即 `_Grunnable`；
- 运行中：Goroutine 正在某个线程上运行，即 `_Grunning`；

##### M（Machine）

​	M 是操作系统线程。调度器最多可以创建 10000 个线程，但是其中大多数的线程都不会执行用户代码（可能陷入系统调用），最多只会有 `GOMAXPROCS` 个活跃线程能够正常运行。

​	在默认情况下，运行时会将 `GOMAXPROCS` 设置成当前机器的核数，我们也可以在程序中使用 [`runtime.GOMAXPROCS`](https://github.com/golang/go/blob/41d8e61a6b9d8f9db912626eb2bbc535e929fefc/src/runtime/debug.go#L16) 来改变最大的活跃线程数。一个四核机器会创建四个活跃的操作系统线程，每一个线程都对应一个运行时中的 [`runtime.m`](https://github.com/golang/go/blob/41d8e61a6b9d8f9db912626eb2bbc535e929fefc/src/runtime/runtime2.go#L486) 结构体。

​	结构体 [`runtime.m`](https://github.com/golang/go/blob/41d8e61a6b9d8f9db912626eb2bbc535e929fefc/src/runtime/runtime2.go#L486) 表示操作系统线程，这个结构体也包含了几十个字段，这里先来了解几个与 Goroutine 相关的字段：

```go
type m struct {
	g0   *g
	curg *g
	...
}
```

​	其中 g0 是持有调度栈的 Goroutine，`curg` 是在当前线程上运行的用户 Goroutine，这也是操作系统线程唯一关心的两个 Goroutine。g0 是一个运行时中比较特殊的 Goroutine，它会深度参与运行时的调度过程，包括 Goroutine 的创建、大内存分配和 CGO 函数的执行。

##### P（Processor）

​	调度器中的处理器 P 是线程和 Goroutine 的中间层，它能提供线程需要的上下文环境，也会负责调度线程上的等待队列，通过处理器 P 的调度，每一个内核线程都能够执行多个 Goroutine，它能在 Goroutine 进行一些 I/O 操作时及时让出计算资源，提高线程的利用率。

​	因为调度器在启动时就会创建 `GOMAXPROCS` 个处理器，所以 Go 语言程序的处理器数量一定会等于 `GOMAXPROCS`，这些处理器会绑定到不同的内核线程上。

[`runtime.p`](https://github.com/golang/go/blob/41d8e61a6b9d8f9db912626eb2bbc535e929fefc/src/runtime/runtime2.go#L576) 是处理器的运行时表示，作为调度器的内部实现。[`runtime.p`](https://github.com/golang/go/blob/41d8e61a6b9d8f9db912626eb2bbc535e929fefc/src/runtime/runtime2.go#L576) 结构体中的状态 `status` 字段会是以下五种中的一种：

| 状态        | 描述                                                         |
| ----------- | ------------------------------------------------------------ |
| `_Pidle`    | 处理器没有运行用户代码或者调度器，被空闲队列或者改变其状态的结构持有，运行队列为空 |
| `_Prunning` | 被线程 M 持有，并且正在执行用户代码或者调度器                |
| `_Psyscall` | 没有执行用户代码，当前线程陷入系统调用                       |
| `_Pgcstop`  | 被线程 M 持有，当前处理器由于垃圾回收被停止                  |
| `_Pdead`    | 当前处理器已经不被使用                                       |

M 必须拥有 P 才可以执行 G 中的代码，P 含有一个包含多个 G 的队列，P 可以调度 G 交由 M 执行。

#### Goroutine 调度策略

- 队列轮转：P 会周期性的将 G 调度到 M 中执行，执行一段时间后，保存上下文，将 G 放到队列尾部，然后从队列中再取出一个 G 进行调度。除此之外，P 还会周期性的查看全局队列是否有 G 等待调度到 M 中执行。
- 系统调用：当 G0 即将进入系统调用时，M0 将释放 P，进而某个空闲的 M1 获取 P，继续执行 P 队列中剩下的 G。M1 的来源有可能是 M 的缓存池，也可能是新建的。
- 当 G0 系统调用结束后，如果有空闲的 P，则获取一个 P，继续执行 G0。如果没有，则将 G0 放入全局队列，等待被其他的 P 调度。然后 M0 将进入缓存池睡眠。

![](/images/Go基础知识点/GMP-2.png)


### Chan

![](/images/Go基础知识点/chan-1.png)


​	虽然在 Go 语言中也能使用共享内存加互斥锁进行通信，但是 Go 语言提供了一种不同的并发模型，即通信顺序进程（Communicating sequential processes，CSP）[1](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-channel/#fn:1)。Goroutine 和 Channel 分别对应 CSP 中的实体和传递信息的媒介，Goroutine 之间会通过 Channel 传递数据

​	Channel 在运行时使用 [`runtime.hchan`](https://github.com/golang/go/blob/41d8e61a6b9d8f9db912626eb2bbc535e929fefc/src/runtime/chan.go#L32) 结构体表示。我们在 Go 语言中创建新的 Channel 时，实际上创建的都是如下所示的结构：

```go
type hchan struct {
	qcount   uint  // 队列中的总元素个数
   dataqsiz uint  // 环形队列大小，即可存放元素的个数
   buf      unsafe.Pointer // 环形队列指针
   elemsize uint16  //每个元素的大小
   closed   uint32  //标识关闭状态
   elemtype *_type // 元素类型
   sendx    uint   // 发送索引，元素写入时存放到队列中的位置
   recvx    uint   // 接收索引，元素从队列的该位置读出
   recvq    waitq  // 等待读消息的goroutine队列
   sendq    waitq  // 等待写消息的goroutine队列
   lock mutex  //互斥锁，chan不允许并发读写
}
```

[`runtime.hchan`](https://github.com/golang/go/blob/41d8e61a6b9d8f9db912626eb2bbc535e929fefc/src/runtime/chan.go#L32) 结构体中的五个字段 `qcount`、`dataqsiz`、`buf`、`sendx`、`recv` 构建底层的循环队列：

- `qcount` — Channel 中的元素个数；
- `dataqsiz` — Channel 中的循环队列的长度；
- `buf` — Channel 的缓冲区数据指针；
- `sendx` — Channel 的发送操作处理到的位置；
- `recvx` — Channel 的接收操作处理到的位置；

除此之外，`elemsize` 和 `elemtype` 分别表示当前 Channel 能够收发的元素类型和大小；`sendq` 和 `recvq` 存储了当前 Channel 由于缓冲区空间不足而阻塞的 Goroutine 列表，这些等待队列使用双向链表 [`runtime.waitq`](https://github.com/golang/go/blob/41d8e61a6b9d8f9db912626eb2bbc535e929fefc/src/runtime/chan.go#L53) 表示，链表中所有的元素都是 [`runtime.sudog`](https://github.com/golang/go/blob/41d8e61a6b9d8f9db912626eb2bbc535e929fefc/src/runtime/runtime2.go#L345) 结构：

```go
type waitq struct {
	first *sudog
	last  *sudog
}
```

[`runtime.sudog`](https://github.com/golang/go/blob/41d8e61a6b9d8f9db912626eb2bbc535e929fefc/src/runtime/runtime2.go#L345) 表示一个在等待列表中的 Goroutine，该结构中存储了两个分别指向前后 [`runtime.sudog`](https://github.com/golang/go/blob/41d8e61a6b9d8f9db912626eb2bbc535e929fefc/src/runtime/runtime2.go#L345) 的指针以构成链表。

#### 读写流程

##### 向 channel 写数据:

1. 若等待接收队列 recvq 不为空，则缓冲区中无数据或无缓冲区，将直接从 recvq 取出 G ，并把数据写入，最后把该 G 唤醒，结束发送过程。
2. 若缓冲区中有空余位置，则将数据写入缓冲区，结束发送过程。
3. 若缓冲区中没有空余位置，则将发送数据写入 G，将当前 G 加入 sendq ，进入睡眠，等待被读 goroutine 唤醒。

##### 从 channel 读数据

1. 若等待发送队列 sendq 不为空，且没有缓冲区，直接从 sendq 中取出 G ，把 G 中数据读出，最后把 G 唤醒，结束读取过程。
2. 如果等待发送队列 sendq 不为空，说明缓冲区已满，从缓冲区中首部读出数据，把 G 中数据写入缓冲区尾部，把 G 唤醒，结束读取过程。
3. 如果缓冲区中有数据，则从缓冲区取出数据，结束读取过程。
4. 将当前 goroutine 加入 recvq ，进入睡眠，等待被写 goroutine 唤醒。

##### 关闭 channel

关闭 channel 时会将 recvq 中的 G 全部唤醒，本该写入 G 的数据位置为 nil。将 sendq 中的 G 全部唤醒，但是这些 G 会 panic。

panic 出现的场景还有：

1. 关闭值为 nil 的 channel
2. 关闭已经关闭的 channel
3. 向已经关闭的 channel 中写数据


#### 无缓冲 Chan 的发送和接收是否同步
需要同步 
channel 无缓冲时，发送阻塞直到数据被接收，接收阻塞直到读到数据；channel有缓冲时，当缓冲满时发送阻塞，当缓冲空时接收阻塞。

### Context

​	上下文 [`context.Context`](https://github.com/golang/go/blob/41d8e61a6b9d8f9db912626eb2bbc535e929fefc/src/context/context.go#L62) Go 语言中用来设置截止日期、同步信号，传递请求相关值的结构体。Context 只定义了接口，凡是实现该接口的类都可称为是一种 cont。xt

  用途：常用的并发控制技术 ，它可以控制一组呈树状结构的goroutine，每个goroutine拥有相同的上下文。Context 是并发安全的，主要是用于控制多个协程之间的协作、取消操作。

​	 [`context.Context`](https://github.com/golang/go/blob/41d8e61a6b9d8f9db912626eb2bbc535e929fefc/src/context/context.go#L62) 接口定义了四个需要实现的方法，其中包括：

1. `Deadline` — 返回 [`context.Context`](https://github.com/golang/go/blob/41d8e61a6b9d8f9db912626eb2bbc535e929fefc/src/context/context.go#L62) 被取消的时间，也就是完成工作的截止日期；
2. `Done` — 返回一个 Channel，这个 Channel 会在当前工作完成或者上下文被取消后关闭，多次调用 `Done` 方法会返回同一个 Channel；
3. `Err` — 返回 [`context.Context`](https://github.com/golang/go/blob/41d8e61a6b9d8f9db912626eb2bbc535e929fefc/src/context/context.go#L62) 结束的原因，它只会在 `Done` 方法对应的 Channel 关闭时返回非空的值；
   1. 如果 [`context.Context`](https://github.com/golang/go/blob/41d8e61a6b9d8f9db912626eb2bbc535e929fefc/src/context/context.go#L62) 被取消，会返回 `Canceled` 错误；
   2. 如果 [`context.Context`](https://github.com/golang/go/blob/41d8e61a6b9d8f9db912626eb2bbc535e929fefc/src/context/context.go#L62) 超时，会返回 `DeadlineExceeded` 错误；
4. `Value` — 从 [`context.Context`](https://github.com/golang/go/blob/41d8e61a6b9d8f9db912626eb2bbc535e929fefc/src/context/context.go#L62) 中获取键对应的值，对于同一个上下文来说，多次调用 `Value` 并传入相同的 `Key` 会返回相同的结果，该方法可以用来传递请求特定的数据；

```go
type Context interface {
	Deadline() (deadline time.Time, ok bool)
	Done() <-chan struct{}
	Err() error
	Value(key interface{}) interface{}
}
```

​	同时[`context`](https://github.com/golang/go/tree/master/src/context) 包中提供的 `context.Background`、`context.TODO`、`context.WithDeadline` 和 `context.WithValue`函数会返回实现该接口的私有结构体

#### 默认上下文

​	`context`包中最常用的方法还是 `context.Background`、`context.TODO`，这两个方法都会返回预先初始化好的私有变量 `background` 和 `todo`，它们会在同一个 Go 程序中被复用：

```go
var (
	background = new(emptyCtx)
	todo       = new(emptyCtx)
)

func Background() Context {
	return background
}

func TODO() Context {
	return todo
}
```

​	这两个私有变量都是通过 `new(emptyCtx)` 语句初始化的，它们是指向私有结构体 [`context.emptyCtx`](https://github.com/golang/go/blob/41d8e61a6b9d8f9db912626eb2bbc535e929fefc/src/context/context.go#L171) 的指针

​	从源代码来看，`context.Background`和 `context.TODO`也只是互为别名，没有太大的差别，只是在使用和语义上稍有不同：

- `context.Background` 是上下文的默认值，所有其他的上下文都应该从它衍生出来；
- `context.TODO`应该仅在不确定应该使用哪种上下文时使用；

在多数情况下，如果当前函数没有上下文作为入参，我们都会使用 `context.Background`作为起始的上下文向下传递。

![](/images/Go基础知识点/context-1.png)

#### 官方的 context 类型

主要有四种，分别是 `emptyCtx`，`cancelCtx`，`timerCtx`，`valueCtx`：

- emptyCtx：空的 context，实现了上面的 4 个接口，但都是直接 return 默认值，没有具体功能代码。

  ```go
  // An emptyCtx is never canceled, has no values, and has no deadline. It is not
  // struct{}, since vars of this type must have distinct addresses.
  type emptyCtx int
  
  func (*emptyCtx) Deadline() (deadline time.Time, ok bool) {
     return
  }
  
  func (*emptyCtx) Done() <-chan struct{} {
     return nil
  }
  
  func (*emptyCtx) Err() error {
     return nil
  }
  
  func (*emptyCtx) Value(key interface{}) interface{} {
     return nil
  }
  
  func (e *emptyCtx) String() string {
     switch e {
     case background:
        return "context.Background"
     case todo:
        return "context.TODO"
     }
     return "unknown empty Context"
  }
  ```

- cancelCtx：用来取消通知用的 context

  ```go
  // A canceler is a context type that can be canceled directly. The
  // implementations are *cancelCtx and *timerCtx.
  type canceler interface {
     cancel(removeFromParent bool, err error)
     Done() <-chan struct{}
  }
  
  // closedchan is a reusable closed channel.
  var closedchan = make(chan struct{})
  
  func init() {
     close(closedchan)
  }
  
  // A cancelCtx can be canceled. When canceled, it also cancels any children
  // that implement canceler.
  type cancelCtx struct {
     Context
  
     mu       sync.Mutex            // protects following fields
     done     chan struct{}         // created lazily, closed by first cancel call
     children map[canceler]struct{} // set to nil by the first cancel call
     err      error                 // set to non-nil by the first cancel call
  }
  
  func (c *cancelCtx) Value(key interface{}) interface{} {
     if key == &cancelCtxKey {
        return c
     }
     return c.Context.Value(key)
  }
  
  func (c *cancelCtx) Done() <-chan struct{} {
     c.mu.Lock()
     if c.done == nil {
        c.done = make(chan struct{})
     }
     d := c.done
     c.mu.Unlock()
     return d
  }
  
  func (c *cancelCtx) Err() error {
     c.mu.Lock()
     err := c.err
     c.mu.Unlock()
     return err
  }
  
  type stringer interface {
     String() string
  }
  
  func contextName(c Context) string {
     if s, ok := c.(stringer); ok {
        return s.String()
     }
     return reflectlite.TypeOf(c).String()
  }
  
  func (c *cancelCtx) String() string {
     return contextName(c.Context) + ".WithCancel"
  }
  
  // cancel closes c.done, cancels each of c's children, and, if
  // removeFromParent is true, removes c from its parent's children.
  func (c *cancelCtx) cancel(removeFromParent bool, err error) {
     if err == nil {
        panic("context: internal error: missing cancel error")
     }
     c.mu.Lock()
     if c.err != nil {
        c.mu.Unlock()
        return // already canceled
     }
     c.err = err
     if c.done == nil {
        c.done = closedchan
     } else {
        close(c.done)
     }
     for child := range c.children {
        // NOTE: acquiring the child's lock while holding parent's lock.
        child.cancel(false, err)
     }
     c.children = nil
     c.mu.Unlock()
  
     if removeFromParent {
        removeChild(c.Context, c)
     }
  }
  ```

- timerCtx：用来超时通知用的 context

  ```go
  / A timerCtx carries a timer and a deadline. It embeds a cancelCtx to
  // implement Done and Err. It implements cancel by stopping its timer then
  // delegating to cancelCtx.cancel.
  type timerCtx struct {
     cancelCtx
     timer *time.Timer // Under cancelCtx.mu.
  
     deadline time.Time
  }
  
  func (c *timerCtx) Deadline() (deadline time.Time, ok bool) {
     return c.deadline, true
  }
  
  func (c *timerCtx) String() string {
     return contextName(c.cancelCtx.Context) + ".WithDeadline(" +
        c.deadline.String() + " [" +
        time.Until(c.deadline).String() + "])"
  }
  
  func (c *timerCtx) cancel(removeFromParent bool, err error) {
     c.cancelCtx.cancel(false, err)
     if removeFromParent {
        // Remove this timerCtx from its parent cancelCtx's children.
        removeChild(c.cancelCtx.Context, c)
     }
     c.mu.Lock()
     if c.timer != nil {
        c.timer.Stop()
        c.timer = nil
     }
     c.mu.Unlock()
  }
  ```

- valueCtx：用来传值的 context

  ```go
  // A valueCtx carries a key-value pair. It implements Value for that key and
  // delegates all other calls to the embedded Context.
  type valueCtx struct {
     Context
     key, val interface{}
  }
  
  // stringify tries a bit to stringify v, without using fmt, since we don't
  // want context depending on the unicode tables. This is only used by
  // *valueCtx.String().
  func stringify(v interface{}) string {
     switch s := v.(type) {
     case stringer:
        return s.String()
     case string:
        return s
     }
     return "<not Stringer>"
  }
  
  func (c *valueCtx) String() string {
     return contextName(c.Context) + ".WithValue(type " +
        reflectlite.TypeOf(c.key).String() +
        ", val " + stringify(c.val) + ")"
  }
  
  func (c *valueCtx) Value(key interface{}) interface{} {
     if c.key == key {
        return c.val
     }
     return c.Context.Value(key)
  }
  ```

#### 创建emptyCtx，cancelCtx，timerCtx，valueCtx方法

- emptyCtx 

  ```go
  // Background returns a non-nil, empty Context. It is never canceled, has no
  // values, and has no deadline. It is typically used by the main function,
  // initialization, and tests, and as the top-level Context for incoming
  // requests.
  func Background() Context {
     return background
  }
  
  // TODO returns a non-nil, empty Context. Code should use context.TODO when
  // it's unclear which Context to use or it is not yet available (because the
  // surrounding function has not yet been extended to accept a Context
  // parameter).
  func TODO() Context {
     return todo
  }
  ```

- cancelCtx 

  ```go
  // WithCancel returns a copy of parent with a new Done channel. The returned
  // context's Done channel is closed when the returned cancel function is called
  // or when the parent context's Done channel is closed, whichever happens first.
  //
  // Canceling this context releases resources associated with it, so code should
  // call cancel as soon as the operations running in this Context complete.
  func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
     if parent == nil {
        panic("cannot create context from nil parent")
     }
     c := newCancelCtx(parent)
     propagateCancel(parent, &c)
     return &c, func() { c.cancel(true, Canceled) }
  }
  
  
  ```

- timerCtx 

  ```go
  // WithDeadline returns a copy of the parent context with the deadline adjusted
  // to be no later than d. If the parent's deadline is already earlier than d,
  // WithDeadline(parent, d) is semantically equivalent to parent. The returned
  // context's Done channel is closed when the deadline expires, when the returned
  // cancel function is called, or when the parent context's Done channel is
  // closed, whichever happens first.
  //
  // Canceling this context releases resources associated with it, so code should
  // call cancel as soon as the operations running in this Context complete.
  func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {
     if parent == nil {
        panic("cannot create context from nil parent")
     }
     if cur, ok := parent.Deadline(); ok && cur.Before(d) {
        // The current deadline is already sooner than the new one.
        return WithCancel(parent)
     }
     c := &timerCtx{
        cancelCtx: newCancelCtx(parent),
        deadline:  d,
     }
     propagateCancel(parent, c)
     dur := time.Until(d)
     if dur <= 0 {
        c.cancel(true, DeadlineExceeded) // deadline has already passed
        return c, func() { c.cancel(false, Canceled) }
     }
     c.mu.Lock()
     defer c.mu.Unlock()
     if c.err == nil {
        c.timer = time.AfterFunc(dur, func() {
           c.cancel(true, DeadlineExceeded)
        })
     }
     return c, func() { c.cancel(true, Canceled) }
  }
  ```

- valueCtx 

  ```go
  // WithValue returns a copy of parent in which the value associated with key is
  // val.
  //
  // Use context Values only for request-scoped data that transits processes and
  // APIs, not for passing optional parameters to functions.
  //
  // The provided key must be comparable and should not be of type
  // string or any other built-in type to avoid collisions between
  // packages using context. Users of WithValue should define their own
  // types for keys. To avoid allocating when assigning to an
  // interface{}, context keys often have concrete type
  // struct{}. Alternatively, exported context key variables' static
  // type should be a pointer or interface.
  func WithValue(parent Context, key, val interface{}) Context {
     if parent == nil {
        panic("cannot create context from nil parent")
     }
     if key == nil {
        panic("nil key")
     }
     if !reflectlite.TypeOf(key).Comparable() {
        panic("key is not comparable")
     }
     return &valueCtx{parent, key, val}
  }
  ```

### make 和 new

- `make` 的作用是初始化内置的数据结构，也就是我们在前面提到的切片slice、哈希表map和 Channel
- `new` 的作用是根据传入的类型分配一片内存空间并返回指向这片内存空间的指针

![](/images/Go基础知识点/golang-make-and-new.png)

### 值接收者和指针接收者的区别

​	方法能给用户自定义的类型添加新的行为。它和函数的区别在于方法有一个接收者，给一个函数添加一个接收者，那么它就变成了方法。接收者可以是`值接收者`，也可以是`指针接收者`。

​	在调用方法的时候，值类型既可以调用`值接收者`的方法，也可以调用`指针接收者`的方法；指针类型既可以调用`指针接收者`的方法，也可以调用`值接收者`的方法。

```golang
type Person struct {
	age int
}

func (p Person) howOld() int {
	return p.age
}

func (p *Person) growUp() {
	p.age += 1
}

func main() {
	// qcrao 是值类型
	qcrao := Person{age: 18}

	// 值类型 调用接收者也是值类型的方法
	fmt.Println(qcrao.howOld())

	// 值类型 调用接收者是指针类型的方法
	qcrao.growUp()
	fmt.Println(qcrao.howOld())

	// ----------------------

	// stefno 是指针类型
	stefno := &Person{age: 100}

	// 指针类型 调用接收者是值类型的方法
	fmt.Println(stefno.howOld())

	// 指针类型 调用接收者也是指针类型的方法
	stefno.growUp()
	fmt.Println(stefno.howOld())
}

//output
18
19
100
101
```

| -              | 值接收者                                                     |                          指针接收者                          |
| -------------- | ------------------------------------------------------------ | :----------------------------------------------------------: |
| 值类型调用者   | 方法会使用调用者的一个副本，类似于“传值”                     | 使用值的引用来调用方法，上例中，`qcrao.growUp()` 实际上是 `(&qcrao).growUp()` |
| 指针类型调用者 | 指针被解引用为值，上例中，`stefno.howOld()` 实际上是 `(*stefno).howOld()` | 实际上也是“传值”，方法里的操作会影响到调用者，类似于指针传参，拷贝了一份指针 |

实现了接收者是值类型的方法，相当于自动实现了接收者是指针类型的方法；而实现了接收者是指针类型的方法，不会自动生成对应接收者是值类型的方法。

```golang
type coder interface {
	code()
	debug()
}

type Gopher struct {
	language string
}

func (p Gopher) code() {
	fmt.Printf("I am coding %s language\n", p.language)
}

func (p *Gopher) debug() {
	fmt.Printf("I am debuging %s language\n", p.language)
}

func main() {
	var c coder = &Gopher{"Go"}
	c.code()
	c.debug()
    
    //报错
   	var c1 coder = Gopher{"Go"}
	c1.code()
	c1.debug()
}
```

所以，当实现了一个接收者是值类型的方法，就可以自动生成一个接收者是对应指针类型的方法，因为两者都不会影响接收者。但是，当实现了一个接收者是指针类型的方法，如果此时自动生成一个接收者是值类型的方法，原本期望对接收者的改变（通过指针实现），现在无法实现，因为值类型会产生一个拷贝，不会真正影响调用者。

最后，只要记住下面这点就可以了：

> 如果实现了接收者是值类型的方法，会隐含地也实现了接收者是指针类型的方法。

#### 两者分别在何时使用 

如果方法的接收者是值类型，无论调用者是对象还是对象指针，修改的都是对象的副本，不影响调用者；如果方法的接收者是指针类型，则调用者修改的是指针指向的对象本身。

使用指针作为方法的接收者的理由：

- 方法能够修改接收者指向的值。
- 避免在每次调用方法时复制该值，在值的类型为大型结构体时，这样做会更加高效。

是使用值接收者还是指针接收者，不是由该方法是否修改了调用者（也就是接收者）来决定，而是应该基于该类型的`本质`。

如果类型具备“原始的本质”，也就是说它的成员都是由 Go 语言里内置的原始类型，如字符串，整型值等，那就定义值接收者类型的方法。像内置的引用类型，如 slice，map，interface，channel，这些类型比较特殊，声明他们的时候，实际上是创建了一个 `header`， 对于他们也是直接定义值接收者类型的方法。这样，调用函数时，是直接 copy 了这些类型的 `header`，而 `header` 本身就是为复制设计的。

如果类型具备非原始的本质，不能被安全地复制，这种类型总是应该被共享，那就定义指针接收者的方法。比如 go 源码里的文件结构体（struct File）就不应该被复制，应该只有一份`实体`。

### 接口的动态类型和动态值

接口值的零值是指`动态类型`和`动态值`都为 `nil`。当仅且当这两部分的值都为 `nil` 的情况下，这个接口值就才会被认为 `接口值 == nil`

### 反射

维基百科上反射的定义：

> 在计算机科学中，反射是指计算机程序在运行时（Run time）可以访问、检测和修改它本身状态或行为的一种能力。用比喻来说，反射就是程序在运行的时候能够“观察”并且修改自己的行为。

《Go 语言圣经》中是这样定义反射的：

> Go 语言提供了一种机制在运行时更新变量和检查它们的值、调用它们的方法，但是在编译时并不知道这些变量的具体类型，这称为反射机制。

使用反射的常见场景有以下两种：

1. 不能明确接口调用哪个函数，需要根据传入的参数在运行时决定。
2. 不能明确传入函数的参数类型，需要在运行时处理任意对象。

#### 如何实现

​	`interface`，它是 Go 语言实现抽象的一个非常强大的工具。当向接口变量赋予一个实体类型的时候，接口会存储实体的类型信息，反射就是通过接口的类型信息实现的，反射建立在类型的基础上

​	Go 语言中，每个变量都有一个静态类型，在编译阶段就确定了的，比如 `int, float64, []int` 等等。注意，这个类型是声明时候的类型，不是底层数据类型。

```golang
type MyInt int

var i int
var j MyInt
尽管 i，j 的底层类型都是 int，他们是不同的静态类型，除非进行类型转换，否则，i 和 j 不能同时出现在等号两侧。j 的静态类型就是 MyInt
```

反射主要与 interface{} 类型相关。关于 interface 的底层结构（iface和eface），如下所示

```golang
type iface struct {
	tab  *itab
	data unsafe.Pointer
}

type itab struct {
	inter  *interfacetype
	_type  *_type
	link   *itab
	hash   uint32
	bad    bool
	inhash bool
	unused [2]byte
	fun    [1]uintptr
}
```

其中 `itab` 由具体类型 `_type` 以及 `interfacetype` 组成。`_type` 表示具体类型，而 `interfacetype` 则表示具体类型实现的接口类型

![](/images/Go基础知识点/reflect-1.png)

实际上，iface 描述的是非空接口，它包含方法；与之相对的是 `eface`，描述的是空接口，不包含任何方法，Go 语言里所以类型都 `“实现了”` 空接口

```golang
type eface struct {
    _type *_type
    data  unsafe.Pointer
}
```

相比 `iface`，`eface` 就比较简单了。只维护了一个 `_type` 字段，表示空接口所承载的具体的实体类型。`data` 描述了具体的值

![](/images/Go基础知识点/reflect-2.png)

#### 接口interface的静态类型和动态类型

```golang
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}



var r io.Reader
tty, err := os.OpenFile("/Users/qcrao/Desktop/test", os.O_RDWR, 0)
if err != nil {
    return nil, err
}
r = tty
```

首先声明 `r` 的类型是 `io.Reader`，注意，这是 `r` 的静态类型，此时它的动态类型为 `nil`，并且它的动态值也是 `nil`。

之后，`r = tty` 这一语句，将 `r` 的动态类型变成 `*os.File`，动态值则变成非空，表示打开的文件对象。这时，r 可以用`<value, type>`对来表示为： `<tty, *os.File>`

![](/images/Go基础知识点/reflect-3.png)

注意看上图，此时虽然 `fun` 所指向的函数只有一个 `Read` 函数，其实 `*os.File` 还包含 `Write` 函数，也就是说 `*os.File` 其实还实现了 `io.Writer` 接口。因此下面的断言语句可以执行：

```golang
var w io.Writer
w = r.(io.Writer)
```

之所以用断言，而不能直接赋值，是因为 `r` 的静态类型是 `io.Reader`，并没有实现 `io.Writer` 接口。断言能否成功，看 `r` 的动态类型是否符合要求

这样，w 也可以表示成 `<tty, *os.File>`，仅管它和 `r` 一样，但是 w 可调用的函数取决于它的静态类型 `io.Writer`，也就是说它只能有这样的调用形式： `w.Write()` 。`w` 的内存形式如下图：

![](/images/Go基础知识点/reflect-4.png)

和 `r` 相比，仅仅是 `fun` 对应的函数变了：`Read -> Write`

接着将w赋值给empty

```golang
var empty interface{}
empty = w
```

![](/images/Go基础知识点/reflect-5.png)

![](/images/Go基础知识点/reflect-6.png)

#### 反射的基本函数

- reflect.ValueOf() 获取输入参数接口中的数据的值

  源码：src/reflect/value.go

  ```golang
  // ValueOf returns a new Value initialized to the concrete value
  // stored in the interface i. ValueOf(nil) returns the zero Value.
  func ValueOf(i interface{}) Value {
  	if i == nil {
  		return Value{}
  	}
  
  	// TODO: Maybe allow contents of a Value to live on the stack.
  	// For now we make the contents always escape to the heap. It
  	// makes life easier in a few places (see chanrecv/mapassign
  	// comment below).
  	escapes(i)
  
  	return unpackEface(i)
  }
  
  还有一堆关于value的类型方法.......
  ```

- reflect.TypeOf() 获取输入参数接口中值的类型

  源码：src/reflect/type.go

  ```golang
  func TypeOf(i interface{}) Type {
  	eface := *(*emptyInterface)(unsafe.Pointer(&i))
  	return toType(eface.typ)
  }
  
  type Type interface {
      // 所有的类型都可以调用下面这些函数
  
  	// 此类型的变量对齐后所占用的字节数
  	Align() int
  	
  	// 如果是 struct 的字段，对齐后占用的字节数
  	FieldAlign() int
  
  	// 返回类型方法集里的第 `i` (传入的参数)个方法
  	Method(int) Method
  
  	// 通过名称获取方法
  	MethodByName(string) (Method, bool)
  
  	// 获取类型方法集里导出的方法个数
  	NumMethod() int
  
  	// 类型名称
  	Name() string
  
  	// 返回类型所在的路径，如：encoding/base64
  	PkgPath() string
  
  	// 返回类型的大小，和 unsafe.Sizeof 功能类似
  	Size() uintptr
  
  	// 返回类型的字符串表示形式
  	String() string
  
  	// 返回类型的类型值
  	Kind() Kind
  
  	// 类型是否实现了接口 u
  	Implements(u Type) bool
  
  	// 是否可以赋值给 u
  	AssignableTo(u Type) bool
  
  	// 是否可以类型转换成 u
  	ConvertibleTo(u Type) bool
  
  	// 类型是否可以比较
  	Comparable() bool
  
  	// 下面这些函数只有特定类型可以调用
  	// 如：Key, Elem 两个方法就只能是 Map 类型才能调用
  	
  	// 类型所占据的位数
  	Bits() int
  
  	// 返回通道的方向，只能是 chan 类型调用
  	ChanDir() ChanDir
  
  	// 返回类型是否是可变参数，只能是 func 类型调用
  	// 比如 t 是类型 func(x int, y ... float64)
  	// 那么 t.IsVariadic() == true
  	IsVariadic() bool
  
  	// 返回内部子元素类型，只能由类型 Array, Chan, Map, Ptr, or Slice 调用
  	Elem() Type
  
  	// 返回结构体类型的第 i 个字段，只能是结构体类型调用
  	// 如果 i 超过了总字段数，就会 panic
  	Field(i int) StructField
  
  	// 返回嵌套的结构体的字段
  	FieldByIndex(index []int) StructField
  
  	// 通过字段名称获取字段
  	FieldByName(name string) (StructField, bool)
  
  	// FieldByNameFunc returns the struct field with a name
  	// 返回名称符合 func 函数的字段
  	FieldByNameFunc(match func(string) bool) (StructField, bool)
  
  	// 获取函数类型的第 i 个参数的类型
  	In(i int) Type
  
  	// 返回 map 的 key 类型，只能由类型 map 调用
  	Key() Type
  
  	// 返回 Array 的长度，只能由类型 Array 调用
  	Len() int
  
  	// 返回类型字段的数量，只能由类型 Struct 调用
  	NumField() int
  
  	// 返回函数类型的输入参数个数
  	NumIn() int
  
  	// 返回函数类型的返回值个数
  	NumOut() int
  
  	// 返回函数类型的第 i 个值的类型
  	Out(i int) Type
  
      // 返回类型结构体的相同部分
  	common() *rtype
  	
  	// 返回类型结构体的不同部分
  	uncommon() *uncommonType
  }
  ```

- 
  reflect.ValueOf().Elem() 获取原始可操作的数据

#### 反射的三大定律 

- 反射可以将interface 类型变量转换成反射对象

  ```go
  func main(){
     var x float64 = 3.4
     t := reflect.TypeOf(x)
     fmt.Println("type:", t)
  
     v := reflect.ValueOf(x)
     fmt.Println("value", v)
  }
  ```

- 反射可以将反射对象还原成interface 对象

  ```go
  func main(){
     var x float64 = 3.4
     v := reflect.ValueOf(x) //v is reflext.Value
     var y float64 = v.Interface().(float64)
     fmt.Println("value", y)
  }
  ```

- 反射对象可修改，value值必须是可设置的

  ```
  func main(){
     var x float64 = 3.4
     v := reflect.ValueOf(&x)
     v.Elem().SetFloat(6.6)
     fmt.Println("x :", v.Elem().Interface())
  }
  ```

### 断言



类型断言（Type Assertion）是一个使用在接口值上的操作，用于检查接口类型变量所持有的值是否实现了期望的接口或者具体的类型。

在Go语言中类型断言的语法格式如下：

```go
value, ok := x.(T)
```

其中，x 表示一个接口的类型，T 表示一个具体的类型（也可为接口类型）。

该断言表达式会返回 x 的值（也就是 value）和一个布尔值（也就是 ok），可根据该布尔值判断 x 是否为 T 类型：

- 如果 T 是具体某个类型，类型断言会检查 x 的动态类型是否等于具体类型 T。如果检查成功，类型断言返回的结果是 x 的**动态值**，其类型是 T。
- 如果 T 是接口类型，类型断言会检查 x 的动态类型是否满足 T。如果检查成功，x 的动态值不会被提取，返回值是一个类型为 T 的接口值。
- 无论 T 是什么类型，如果 x 是 nil 接口值，类型断言都会失败。

### 内存模型

​	基本模型的相关概念和理解可以参考我另一篇文章（重点看图理解）：https://mrvwy.github.io/2020/08/02/%E8%81%8A%E8%81%8Athread-caching-malloc/

![](/images/Go基础知识点/20200722022116565.png)

​	 [TCMalloc](https://github.com/google/tcmalloc)是用来替代传统的malloc内存分配函数。它有减少内存碎片，适用于多核，更好的并行性支持等特性。

相关名词概念：

1. Page

   操作系统对内存管理的单位，TCMalloc也是以页为单位管理内存，但是TCMalloc中Page大小是操作系统中页的倍数关系。2，4，8 ....

2. Span

   Span 是PageHeap中管理内存页的单位，它是由一组连续的Page组成，比如2个Page组成的span，多个这样的span就用链表来管理。当然，还可以有4个Page组成的span等等。

3. ThreadCache

   ThreadCache是每个线程各自独立拥有的cache，一个cache包含多个空闲内存链表（size classes），每一个链表（size-class）都有自己的object，每个object都是大小相同的。

4. CentralCache

   CentralCache是当ThreadCache内存不足时，提供内存供其使用。它保持的是空闲块链表，链表数量和ThreadCache数量相同。ThreadCache中内存过多时，可以放回CentralCache中。

5. PageHeap

   PageHeap保存的也是若干链表，不过链表保存的是Span（多个相同的page组成一个Span）。CentralCache内存不足时，可以从PageHeap获取Span，然后把Span切割成object。

### map

map底层是基于哈希表+链地址法存储的

#### 为啥是无序的

```go
//https://github.com/golang/go/blob/36f30ba289e31df033d100b2adb4eaf557f05a34/src/runtime/map.go
func mapiterinit(t *maptype, h *hmap, it *hiter) {

	// decide where to start
	r := uintptr(fastrand())
	if h.B > 31-bucketCntBits {
		r += uintptr(fastrand()) << 31
	}

	mapiternext(it)
}
```

主要是因为 map 在扩容后，可能会将部分 key 移至新内存，那么这一部分实际上就已经是无序的了。而遍历的过程，其实就是按顺序遍历内存地址，同时按顺序遍历内存地址中的 key。但这时已经是无序的了。当然如果就一个 map，我保证不会对 map 进行修改删除等操作，那么按理说没有扩容就不会发生改变。但也是因为这样，GO 才在源码中加上随机的元素，将遍历 map 的顺序随机化，用来防止使用者用来顺序遍历，因为在某些情况下，可能会酿成大错。

#### 原理

```go
// A header for a Go map.
type hmap struct {
   // 元素个数，调用 len(map) 时，直接返回此值
   count     int
   flags     uint8
   // buckets 的对数 log_2
   B         uint8
   // overflow 的 bucket 近似数
   noverflow uint16
   // 计算 key 的哈希的时候会传入哈希函数
   hash0     uint32
   // 指向 buckets 数组，大小为 2^B
   // 如果元素个数为0，就为 nil
   buckets    unsafe.Pointer
   // 扩容的时候，buckets 长度会是 oldbuckets 的两倍
   oldbuckets unsafe.Pointer
   // 指示扩容进度，小于此地址的 buckets 迁移完成
   nevacuate  uintptr
   extra *mapextra // optional fields
}
```

`B` 是 buckets 数组的长度的对数，也就是说 buckets 数组的长度就是 2^B。bucket 里面存储了 key 和 value，后面会再讲。

buckets 是一个指针，最终它指向的是一个结构体：

```go
// A bucket for a Go map.
type bmap struct {
   // tophash generally contains the top byte of the hash value
   // for each key in this bucket. If tophash[0] < minTopHash,
   // tophash[0] is a bucket evacuation state instead.
   tophash [bucketCnt]uint8
   // Followed by bucketCnt keys and then bucketCnt elems.
   // NOTE: packing all the keys together and then all the elems together makes the
   // code a bit more complicated than alternating key/elem/key/elem/... but it allows
   // us to eliminate padding which would be needed for, e.g., map[int64]int8.
   // Followed by an overflow pointer.
}
```

`bmap` 就是我们常说的“桶”，桶里面会最多装 8 个 key，这些 key 之所以会落入同一个桶，是因为它们经过哈希计算后，哈希结果是“一类”的。在桶内，又会根据 key 计算出来的 hash 值的高 8 位来决定 key 到底落入桶内的哪个位置（一个桶内最多有8个位置）。

整体图

![](/images/Go基础知识点/map-1.png)

当 map 的 key 和 value 都不是指针，并且 size 都小于 128 字节的情况下，会把 bmap 标记为不含指针，这样可以避免 gc 时扫描整个 hmap。但是，我们看 bmap 其实有一个 overflow 的字段，是指针类型的，破坏了 bmap 不含指针的设想，这时会把 overflow 移动到 extra 字段来。

```go
// mapextra holds fields that are not present on all maps.
type mapextra struct {
   // If both key and elem do not contain pointers and are inline, then we mark bucket
   // type as containing no pointers. This avoids scanning such maps.
   // However, bmap.overflow is a pointer. In order to keep overflow buckets
   // alive, we store pointers to all overflow buckets in hmap.extra.overflow and hmap.extra.oldoverflow.
   // overflow and oldoverflow are only used if key and elem do not contain pointers.
   // overflow contains overflow buckets for hmap.buckets.
   // oldoverflow contains overflow buckets for hmap.oldbuckets.
   // The indirection allows to store a pointer to the slice in hiter.
   overflow    *[]*bmap
   oldoverflow *[]*bmap

   // nextOverflow holds a pointer to a free overflow bucket.
   nextOverflow *bmap
}
```

bmap 是存放 k-v 的地方，bmap 的内部组成如下

![](/images/Go基础知识点/map-1.png)

上图就是 bucket 的内存模型，`HOB Hash` 指的就是 top hash。 注意到 key 和 value 是各自放在一起的，并不是 `key/value/key/value/...` 这样的形式。源码里说明这样的好处是在某些情况下可以省略掉 padding 字段，节省内存空间。

例如，有这样一个类型的 map：map[int64]int8，如果按照 `key/value/key/value/...` 这样的模式存储，那在每一个 key/value 对之后都要额外 padding 7 个字节；而将所有的 key，value 分别绑定到一起，这种形式 `key/key/.../value/value/...`，则只需要在最后添加 padding。每个 bucket 设计成最多只能放 8 个 key-value 对，如果有第 9 个 key-value 落入当前的 bucket，那就需要再构建一个 bucket ，通过 `overflow` 指针连接起来。

#### 扩容

符合下面这 2 个条件，就会触发扩容：

1. 装载因子超过阈值，源码里定义的阈值是 6.5。当装载因子超过 6.5 时，表明很多 bucket 都快要装满了，查找效率和插入效率都变低了。在这个时候进行扩容是有必要的
2. overflow 的 bucket 数量过多：当 B 小于 15，也就是 bucket 总数 2^B 小于 2^15 时，如果 overflow 的 bucket 数量超过 2^B；当 B >= 15，也就是 bucket 总数 2^B 大于等于 2^15，如果 overflow 的 bucket 数量超过 2^15。

#### 注意

1. 直接将使用 map1 == map2 是错误的。这种写法只能比较 map 是否为 nil。
2. 无法对 map 的 key 或 value 进行取址
3. map 并不是一个线程安全的数据结构。同时读写一个 map 是未定义的行为，如果被检测到，会直接 panic。可以通过读写锁来解决：`sync.RWMutex`
4. map 不是线程安全的。在查找、赋值、遍历、删除的过程中都会检测写标志，一旦发现写标志置位（等于1），则直接 panic。赋值和删除函数在检测完写标志是复位之后，先将写标志位置位，才会进行之后的操作。



### 值传递和引用传递

#### 形参和实参

- 形式参数：是在定义函数名和函数体的时候使用的参数,目的是用来接收调用该函数时传入的参数。
- 实际参数：在调用有参函数时，主调函数和被调函数之间有数据传递关系。在主调函数中调用一个函数时，函数名后面括号中的参数称为“实际参数”

#### 值传递和引用传递

Go语言中所有的传参都是值传递（传值），都是一个副本，一个拷贝。因为拷贝的内容有时候是非引用类型（int、string、struct等这些），这样就在函数中就无法修改原内容数据；有的是引用类型（指针、map、slice、chan等这些），这样就可以修改原内容数据。

### 内存对齐

​	相关概念和理解可以参考我另一篇文章：https://mrvwy.github.io/2020/07/18/data-structure-alignment/

​	例子：

![](/images/Go基础知识点/内存对齐.png)

变量 a、b 各占据 3 字节的空间，内存对齐后，a、b 占据 4 字节空间，CPU 读取 b 变量的值只需要进行一次内存访问。如果不进行内存对齐，CPU 读取 b 变量的值需要进行 2 次内存访问。第一次访问得到 b 变量的第 1 个字节，第二次访问得到 b 变量的后两个字节。

### 竞态、内存逃逸
#### 竟态
资源竞争，就是在程序中，同一块内存同时被多个 goroutine 访问。我们使用 go build、go run、go test 命令时，添加 -race 标识可以检查代码中是否存在资源竞争。

解决这个问题，我们可以给资源进行加锁，让其在同一时刻只能被一个协程来操作。例如：sync.Mutex，sync.RWMutex


#### 逃逸
逃逸分析就是程序运行时内存的分配位置(栈或堆)，是由编译器来确定的。堆适合不可预知大小的内存分配。但是为此付出的代价是分配速度较慢，而且会形成内存碎片。

逃逸场景：
1. 指针逃逸
2. 栈空间不足逃逸
3. 动态类型逃逸
4. 闭包引用对象逃逸














### 问题
####  rune 类型

- uint8 类型，或者叫 byte 型，代表了 ASCII 码的一个字符。
- rune 类型，代表一个 UTF-8 字符，当需要处理中文、日文或者其他复合字符时，则需要用到 rune 类型。rune 类型等价于 int32 类型。

#### defer和return执行先后顺序

1. 多个defer的执行顺序为“后进先出”；
2. defer、return、返回值三者的执行逻辑应该是：return最先执行，return负责将结果写入返回值中；接着defer开始执行一些收尾工作；最后函数携带当前返回值退出。

​	如果函数的返回值是无名的（不带命名返回值），则go语言会在执行return的时候会执行一个类似**创建一个临时变量作为保存return值的动作**，而有名返回值的函数，由于返回值在函数定义的时候已经将该变量进行定义，在执行return的时候会先执行返回值保存操作，而后续的defer函数会改变这个返回值(虽然defer是在return之后执行的，但是由于使用的函数定义的变量，所以执行defer操作后对该变量的修改会影响到return的值

```go
//例子1
func increaseA() int { // 返回的是ret
    var i int //1. i被初始化为0
    defer func() {
        i++ // 3. i = 1
    }()
    return i // 2. ret = i
}

func increaseB() (r int) {
    defer func() {
        r++
    }()
    return r
}

func main() {
    fmt.Println(increaseA())
    fmt.Println(increaseB())
}
//output ：  0  1
//分析：1.发生return时，将赋值给返回值（可以理解为go自动创建了一个返回值ret，相当于执行了ret=value），此时defer修改该的是value而不是ret。
//2 .而在命名返回值函数中，由于返回值在方法定义时已经被定义，所以不会在创建retValue了，defer对value的修改也会被直接返回。

```

```go
//例子2
func f1() (r int) { //有命名返回，返回是r
    defer func() {
        r++ //2. r = r + 1 = 1
    }()
    return 0 // 1. r = 0
}

func f2() (r int) { //有命名返回，返回是r
    t := 5
    defer func() {
        t = t + 5 // 2. t = t + 5 = 10
    }() 
    return t // 1 . r = t = 5
}

func f3() (r int) { //有命名返回，返回是r
    defer func(r int) {
        r = r + 5  //进入defer的函数之后r又是一个全新的指向不同内存地址的变量，不会对外层函数返回的r造成影响
    }(r)
    return 1 //1. r = 1
}

func f4() (r int) { //有命名返回，返回是r
    t := 5
    defer func() {
        r = t + 5 // 2. r = t + 5 = 10
    }()
    return t //1. r = t = 5
}
//output ： 1, 5, 1, 10
```

```go
//例子3
var a bool = true
func main() {
    defer func(){
        fmt.Println("1")
    }()
    if a == true {
        fmt.Println("2")
        return
    }
    defer func(){
        fmt.Println("3")
    }()
}
//output ： 2, 1
//分析 ： 关键字后面的函数或者方法想要执行必须先注册，return 之后的 defer 是不能注册的， 也就不能执行后面的函数或方法。
```

```go
//例子4
func calc(index string, a, b int) int {
    ret := a + b
    fmt.Println(index, a, b, ret)
    return ret
}

func main() {
    a := 1
    b := 2
    defer calc("1", a, calc("10", a, b)) 
    a = 0
    defer calc("2", a, calc("20", a, b))
    b = 1 
}
//output : 10 1 2 3
//         20 0 2 2
//         2 0 2 2
//         1 1 3 4
//分析： 不管代码顺序如何，defer-calc-func中参数b必须先计算，故会在运行到第一个defer时，执行calc("10",a,b)输出：10 1 2 3得到值3，将cal("1",1,3)存放到延后执执行函数队列中。同理运行到第二个defer也是一样。
```

#### 多重赋值

```go
func main() {
    i := 1
    s := []string{"A", "B", "C"}
    i, s[i-1] = 2, "Z" // 先计算 s[i-1] 最后变成 i, s[0] = 2, “Z”
    fmt.Printf("s: %v \n", s)
}
//output ： [Z,B,C]
```

多重赋值分为两个步骤，有先后顺序：

- 计算等号左边的索引表达式和取址表达式，接着计算等号右边的表达式
- 赋值

#### nil 可以用作 interface、function、pointer、map、slice 和 channel 的“空值”

​	nil 可以用作 interface、function、pointer、map、slice 和 channel 的“空值”


#### 协程 线程 进程
进程: 进程是具有一定独立功能的程序，进程是系统资源分配和调度的最小单位。 每个进程都有自己的独立内存空间，不同进程通过进程间通信来通信。由于进程比较重量，占据独立的内存，所以上下文进程间的切换开销（栈、寄存器、虚拟内存、文件句柄等）比较大，但相对比较稳定安全。  

线程: 线程是进程的一个实体,线程是内核态,而且是CPU调度和分派的基本单位,它是比进程更小的能独立运行的基本单位。线程间通信主要通过共享内存，上下文切换很快，资源开销较少，但相比进程不够稳定容易丢失数据。  
  
协程: 协程是一种用户态的轻量级线程，协程的调度完全是由用户来控制的。协程拥有自己的寄存器上下文和栈。 协程调度切换时，将寄存器上下文和栈保存到其他地方，在切回来的时候，恢复先前保存的寄存器上下文和栈，直接操作栈则基本没有内核切换的开销，可以不加锁的访问全局变量，所以上下文的切换非常快。  


#### 除了加 Mutex 锁以外还有哪些方式安全读写共享变量？
Goroutine 可以通过 Channel 进行安全读写共享变量。Channel是异步进行的

#### channel 为什么它可以做到线程安全？
Channel 可以理解是一个先进先出的队列，通过管道进行通信,发送一个数据到Channel和从Channel接收一个数据都是原子性的。不要通过共享内存来通信，而是通过通信来共享内存，前者就是传统的加锁，后者就是Channel。设计Channel的主要目的就是在多任务间传递数据的，本身就是安全的。

#### Goroutine和线程的区别？

一个线程可以有多个协程

线程、进程都是同步机制，而协程是异步

协程可以保留上一次调用时的状态，当过程重入时，相当于进入了上一次的调用状态

协程是需要线程来承载运行的，所以协程并不能取代线程，「线程是被分割的CPU资源，协程是组织好的代码流程」

#### Struct能不能比较？

相同struct类型的可以比较

不同struct类型的不可以比较,编译都不过，类型不匹配

#### Slice如何扩容？

在使用 append 向 slice 追加元素时，若 slice 空间不足则会发生扩容，扩容会重新分配一块更大的内存，将原 slice 拷贝到新 slice ，然后返回新 slice。扩容后再将数据追加进去。

扩容操作只对容量，扩容后的 slice 长度不变，容量变化规则如下：

1. 若 slice 容量小于1024个元素，那么扩容的时候slice的cap就翻番，乘以2；一旦元素个数超过1024个元素，增长因子就变成1.25，即每次增加原来容量的四分之一。
2. 若 slice 容量够用，则将新元素追加进去，slice.len++，返回原 slice
3. 若 slice 容量不够用，将 slice 先扩容，扩容得到新 slice，将新元素追加进新 slice，slice.len++，返回新 slice。

#### Go中的map如何实现顺序读取？

Go中map如果要实现顺序读取的话，可以先把map中的key，通过sort包排序。

#### Goroutine发生了泄漏如何检测？

可以通过Go自带的工具pprof或者使用`Gops`去检测诊断当前在系统上运行的Go进程的占用的资源。

#### 函数传参是值类型还是引用类型？

在Go语言中只存在值传递，要么是值的副本，要么是指针的副本。无论是值类型的变量还是引用类型的变量亦或是指针类型的变量作为参数传递都会发生值拷贝，开辟新的内存空间。

另外值传递、引用传递和值类型、引用类型是两个不同的概念，不要混淆了。引用类型作为变量传递可以影响到函数外部是因为发生值拷贝后新旧变量指向了相同的内存地址




### Reference

- https://draveness.me/golang/docs/part3-runtime/ch07-memory/golang-memory-allocator/
- https://golang.design/go-questions/interface/receiver/
- https://github.com/LeoYang90/Golang-Internal-Notes
- https://www.topgoer.cn/docs/gomianshiti



