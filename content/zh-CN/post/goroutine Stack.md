---
title: "Go goroutine Stack"
date: "2020-06-01 22:56:00"
categories:
  - Go
tags:
  - Go
toc: true
math: true
type: about
---

#### 前言

goroutine需要自己的栈才能够运行。假如每个goroutine分配的栈固定，那么这个栈太小则会导致溢出，太大又会浪费空间，因此如果这样的gorouting有成千上万个，那么系统是无法容纳众多的goroutine的。 因此在当前Go版本中， goroutine就已经在使用连续栈的形式来为其分配内存空间，该形式可以让goroutine尽可能的把内存空间最大化利用同时，也能正常运行。

#### Goroutine Stack 的结构体定义

首先我们来看一下在Go的源码中对 Goroutine Stack 的结构体定义：

```go
type stack struct {
   lo uintptr  栈空间的低地址
   hi uintptr  栈空间的高地址
}

// uintptr is an integer type that is large enough to hold the bit pattern of
// any pointer.
```

每当我们起一个 goroutine 时，Go会给这个 goroutine 分配一个内存空间，其大小基于所使用的平台，在源码的定义如下：

```
// Number of orders that get caching. Order 0 is FixedStack
// and each successive order is twice as large.
// We want to cache 2KB, 4KB, 8KB, and 16KB stacks. Larger stacks
// will be allocated directly.
// Since FixedStack is different on different systems, we
// must vary NumStackOrders to keep the same maximum cached size.
//   OS                FixedStack  NumStackOrders
//   -----------------+------------+---------------
//   linux/darwin/bsd  2KB         4
//   windows/32        4KB         3
//   windows/64        8KB         2
//   plan9             4KB         3
```



#### 栈的初始化

首先是对栈的初始化：

```go
func stackinit() {

    if _StackCacheSize&_PageMask != 0 {
        throw("cache size must be a multiple of page size")
    }
            //从栈池中取出一个栈来init()相当对mSpanList初始化
    for i := range stackpool {
        stackpool[i].init()
    }
    for i := range stackLarge.free {
        stackLarge.free[i].init()
    }
}
```

其中对于stackpool栈池，它的定义是一个 `[_NumStackOrders]mSpanList` ,而 `mSpanList` 涉及到Go的内存管理，在这里先不展开。而`stackLarge（// Global pool of large stack spans.）`个人理解是是最大栈空间全局池。spans可以理解为一块内存。

#### 栈的增大

接下来我们看看当 Goroutine 在运行时栈空间需要增加是如何操作的。

```go
func newstack() {
    //获取当前的goroutine
    thisg := getg() 
    // TODO: double check all gp. shouldn't be getg().
    ...........中间代码省略................
    // Allocate a bigger segment and move the stack.
    oldsize := gp.stack.hi - gp.stack.lo
    newsize := oldsize * 2
    if newsize > maxstacksize {
        print("runtime: goroutine stack exceeds ", maxstacksize, "-byte limit\\n")
        throw("stack overflow")
    }

    //修改该协程的状态 由_Grunning 变成_Gcopystack
    casgstatus(gp, _Grunning, _Gcopystack)

    // The concurrent GC will not scan the stack while we are doing the copy since   将当前栈的数据复制到新的栈中
    // the gp is in a Gcopystack status.
    copystack(gp, newsize, true)


    if stackDebug >= 1 {
    	print("stack grow done\\n")
    }
    //修改该协程的状态 由_Gcopystack变成 _Grunning

    casgstatus(gp, _Gcopystack, _Grunning)
    gogo(&gp.sched)
}
```

从源码来看，一个 Goroutine 栈空间增大的过程为，先获取当前的goroutine，更新syscallsp和syscallpc，应该是防止在增大栈空间时系统又回调了这个 Goroutine ，然后中间的代码（看的我一脸懵）和注解，个人的见解为，在 Goroutine 增大栈的过程中，会令 Goroutine 的状态发生改变，这时候GC的什么写屏障和 Goroutine 抢占的问题就会出现，因此这不详细说明。接着就计算新栈的空间，其等于旧栈空间的一倍（`newsize := oldsize * 2`）然后把该协程g的状态由 \_Grunning 变 \_Gcopystack，复制数据到新栈，把 该协程g 恢复原状，进而完成栈增大操作。

#### 栈的复制

接着上述我们看看copystack()是如何操作的。

```go
func copystack(gp *g, newsize uintptr, sync bool) {
    .......
    old := gp.stack
    .......
    used := old.hi - gp.sched.sp

    // 分配新堆栈
    new := stackalloc(uint32(newsize))
    ..........

    // Compute adjustment. 这一步应该是调整旧栈指针
    var adjinfo adjustinfo
    adjinfo.old = old
    adjinfo.delta = new.hi - old.hi

    ..........

    // 将旧栈复制到新位置
    memmove(unsafe.Pointer(new.hi-ncopy), unsafe.Pointer(old.hi-ncopy), ncopy)

    // Adjust remaining structures that have pointers into stacks.
    // We have to do most of these before we traceback the new
    // stack because gentraceback uses them.
    adjustctxt(gp, &adjinfo)
    adjustdefers(gp, &adjinfo)
    adjustpanics(gp, &adjinfo)
    if adjinfo.sghi != 0 {
    adjinfo.sghi += adjinfo.delta
    }

    // Swap out old stack for new one
    gp.stack = new
    gp.stackguard0 = new.lo + _StackGuard // NOTE: might clobber a preempt request
    gp.sched.sp = new.hi - used
    gp.stktopsp += adjinfo.delta

    // 调整新栈中的指针
    gentraceback(^uintptr(0), ^uintptr(0), 0, gp, 0, nil, 0x7fffffff, adjustframe, noescape(unsafe.Pointer(&adjinfo)), 0)

    // 释放旧栈
    if stackPoisonCopy != 0 {
    fillstack(old, 0xfc)
    }
    stackfree(old)
}
```

从源码来看，一个 Goroutine 栈的复制的过程大致为，先计算调整旧栈的指针，然后再把旧栈复制到新栈中，最后调整新栈的指针(**gentraceback**)和释放旧栈。注意复制栈的过程需要的条件为：当前栈的状态一定是 \_Gcopystack 。

#### 栈的缩小

既然有 Goroutine的增栈，那么必然有 Goroutine 的缩栈，下面我们来看一下缩栈的操作。

```go
// Maybe shrink the stack being used by gp.
// Called at garbage collection time.
// gp must be stopped, but the world need not be.
//由上面的注释可以知道,缩栈是在垃圾回收时间做的
func shrinkstack(gp *g) {
    //获取当前g的状态
    gstatus := readgstatus(gp)
    if gp.stack.lo == 0 {
    	throw("missing stack in shrinkstack")
    }
    if gstatus&_Gscan == 0 {
    	throw("bad status in shrinkstack")
    }

    ............

    //计算新栈的newsize,从这里可以看出Go缩栈是缩小到原来的一半

    oldsize := gp.stack.hi - gp.stack.lo
    newsize := oldsize / 2
    // Don't shrink the allocation below the minimum-sized stack
    // allocation.
    if newsize < _FixedStack {
    return
    }
    // Compute how much of the stack is currently in use and only
    // shrink the stack if gp is using less than a quarter of its
    // current stack. The currently used stack includes everything
    // down to the SP plus the stack guard space that ensures
    // there's room for nosplit functions.

    //当已使用的栈占不到总栈的1/4 进行缩容

    avail := gp.stack.hi - gp.stack.lo
    if used := gp.stack.hi - gp.sched.sp + _StackLimit; used >= avail/4 {
    return
    }

    // We can't copy the stack if we're in a syscall.
    // The syscall might have pointers into the stack.


    //判断当前栈是否在被系统调用,如果正在被调用那么syscall可以有指针在当前栈里面

    if gp.syscallsp != 0 {
    return
    }
    if sys.GoosWindows != 0 && gp.m != nil && gp.m.libcallsp != 0 {
    return
    }

    if stackDebug > 0 {
    print("shrinking stack ", oldsize, "->", newsize, "\\n")
    }

    //最后把数据复制到新栈中
    copystack(gp, newsize, false)
}
```

从 Goroutine 缩栈的源码来看，缩栈的大致过程为，先获取当前goroutine的状态进来判断，然后计算newsize，另外如果gp使用少于其四分之一的空间，则缩小堆栈 。检查当前g是否正在被系统调用， 如果当前g正在被系统调用就直接return回去然后等待系统调用结束，最后把数据复制到新栈中。

#### Goroutine Stack Frame

建议先了解 Stack Frame

##### FP，SP，PC ，LR

PC： 程序计数器 SP： 保存栈顶的地址 FP ： 保存栈底的地址 LR ： 连接寄存器的程序计数器

##### Stack frame layout

从Go的源码可以得知Stack frame 的设计如下：

```
// (x86)
// +------------------+
//  args from caller   //调用者的args
// +------------------+ <- frame->argp
//   return address  
// +------------------+
//   caller's BP (\*)  (\*) if framepointer\_enabled && varp < sp
// +------------------+ <- frame->varp
//      locals       
// +------------------+
//   args to callee    //被调用者的args
// +------------------+ <- frame->sp
//
// (arm)
// +------------------+
//  args from caller 
// +------------------+ <- frame->argp
//  caller's retaddr 
// +------------------+ <- frame->varp
//      locals       
// +------------------+
//   args to callee  
// +------------------+
//   return address  
// +------------------+ <- frame->sp
```

同时还要关注一下这个 结构体 stack traces（栈的跟踪）

```go
type stkframe struct {
   fn       funcInfo   // function being run
   pc       uintptr    // program counter within fn
   continpc uintptr    // program counter where execution can continue, or 0 if not
   lr       uintptr    // program counter at caller aka link register
   sp       uintptr    // stack pointer at pc
   fp       uintptr    // stack pointer at caller aka frame pointer
   varp     uintptr    // top of local variables
   argp     uintptr    // pointer to function arguments
   arglen   uintptr    // number of bytes at argp
   argmap   \*bitvector // force use of this argmap
}
```



#### 关于调整新旧栈的指针

从函数gentraceback()开始，由于 gentraceback() 代码量过于庞大，这里我们只关注 PC，SP，FP，LR 和与stkframe 相关 的代码部分。

```
......
var frame stkframe
frame.pc = pc0
frame.sp = sp0

......
//findfunc->findmoduledatap ,他返回时一个moduledata对象，其记录有关可执行文件布局的信息
f := findfunc(frame.pc)
......
frame.fn = f


for n < max {
        ......
        f = frame.fn
        if f.pcsp == 0 {
            // No frame information, must be external function, like race support.
            // See golang.org/issue/13568.
            break
        }
        ......
        if frame.fp == 0 {
            sp := frame.sp
            ......
            //计算FP
            frame.fp = sp + uintptr(funcspdelta(f, frame.pc, &cache))
            if !usesLR {
  // On x86, call instruction pushes return PC before entering new function.
                frame.fp += sys.RegSize
            }
        }
        var flr funcInfo
        if topofstack(f, gp.m != nil && gp == gp.m.g0) {
            ......
        } else if usesLR && f.funcID == funcID\_jmpdefer {
            ......
        } else {
            var lrPtr uintptr
            if usesLR {
                ......
            } else {
                if frame.lr == 0 {
                    lrPtr = frame.fp - sys.RegSize
                    frame.lr = uintptr(\*(\*sys.Uintreg)(unsafe.Pointer(lrPtr)))
                }
            }
            flr = findfunc(frame.lr)
            ......
        }

    //top of local variables
    frame.varp = frame.fp 

    if !usesLR {
        // On x86, call instruction pushes return PC before entering new function.
        frame.varp -= sys.RegSize
    }
    ......
    if framepointer\_enabled && GOARCH == "amd64" && frame.varp > frame.sp {
        frame.varp -= sys.RegSize
    }
    ......
    if callback != nil  printing {
       frame.argp = frame.fp + sys.MinFrameSize
        ......
    }
    ......
    // Unwind to next frame. 退至下一帧
    frame.fn = flr
    frame.pc = frame.lr
    frame.lr = 0
    frame.sp = frame.fp
    frame.fp = 0
    frame.argmap = nil
    ......
}
......
```

到这 goroutine Stack 也说的差不多了

#### 总结

从 Goroutine 栈的实现方式上可以知道，不管栈扩大还是缩小，都会重新申请一块新栈（新的一块内存），然后把旧栈的数据复制到新栈，相当于完整替换了Goroutine使用的物理内存，再借助指针重新指向新栈，便完成了栈大小改变过程。