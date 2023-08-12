---
title: "Context in Go"
date: "2020-06-09 12:15:38"
categories:
  - Go
tags:
  - Go
toc: true
math: true
type: about
---


![](/images/Context in Go/20200609041455723.png)

### Context Interface

```go
type Context interface { 
  //超过时间，该context便死亡
  Deadline() (deadline time.Time, ok bool)
  //当context结束时，返回一个channel
  Done() <-chan struct{}
  //context错误
  Err() error
  //返回上下文所关联的key/value值
  Value(key interface{}) interface{}
}
```



### emptyCtx

这个emptyCtx是一个int类型，它实现了Context Interface里面的所有方法，但是都返回空值，只有一个String()方法来判断其类型来进行相应输出。

```go
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

同时background和todo也是基于这个emptyCtx类new出来的，了解过background的都知道，在使用context时都是先context.Background()一个context来进行后继操作。

```go
var (
   background = new(emptyCtx)
   todo       = new(emptyCtx)
)
```

而background和todo的区别在于，todo是用于你不清楚要使用哪个上下文的情况下。而background通常是由main函数使用，说白了background就是所有context的root，不能被cancel。

### CancelFunc

这个类定义了一个可以返回的cancel函数，它可以cancel当前的context和它的子context，并且把自己从父类的context删除。它提供三个可以cancel的方法。

#### WithCancel

这个方法主要是当上下文操作完成时立即调用结束该context链。当调用父类的cancel时，父类和其子节点就会一起Done()。

```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
   //创建一个新context节点
   c := newCancelCtx(parent)
   //创建context的链路，其主要是创建一个map来存储context的子节点
   propagateCancel(parent, &c)
   return &c, func() { c.cancel(true, Canceled) }
}
```

但是不同的context子类是分布在不同的goroutine，那么它是如何实现当父类Done()，子节点接着Done()。通常首先想到的方法时子类监听一个channel传过来的关闭信号，然后子类收到这个关闭信号之后就结束自身。但具体如何实现的，请看下图。 ![](/images/Context in Go/20200609041008165.png)

#### WithDeadline

这个方法主要是当超过一定时间后，context就自行结束自己和其子节点。同时它也返回一个可以主动cancel的方法

```go
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {
   ........
   c := &timerCtx{
      cancelCtx: newCancelCtx(parent),
      deadline:  d,
   }
   //创建context的链路，其主要是创建一个map来存储context的子节点
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



#### WithTimeout

这个方法也是一个超时控制，内部其实是引用了WithDeadline方法

```go
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
   return WithDeadline(parent, time.Now().Add(timeout))
}
```



### canceler

canceler是一个具有类似于取消器的接口（A canceler is a context type that can be canceled directly），它定义了二个方法。

```go
type canceler interface {
   cancel(removeFromParent bool, err error)
   Done() <-chan struct{}
}
```



### cancelCtx

首先cancelCtx定义了一个结构体，其继承context接口，同时重点注意加粗部分。

```go
type cancelCtx struct {
   Context

  mu       sync.Mutex            // protects following fields
  done     chan struct{}         // created lazily, closed by first cancel call
  children map[canceler]struct{} // set to nil by the first cancel call
  err      error                 // set to non-nil by the first cancel call
}
```

重点关注cancel()函数和加粗部分，具体代码解释就不说了，源码写的很清楚。

```go
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



### timerCtx

首先timerCtx定义了一个结构体，其继承cancelCtx结构

```go
type timerCtx struct {
   cancelCtx
   timer *time.Timer // 计时器，A Timer must be created with NewTimer or AfterFunc.
   deadline time.Time //结束时间
}
```

同时其定义了三个方法分别为Deadline()、String()、cancel()。重点关注cancel()函数

```go
func (c *timerCtx) cancel(removeFromParent bool, err error) {
   c.cancelCtx.cancel(false, err) //cancel当前context和其子节点
   if removeFromParent {
      // Remove this timerCtx from its parent cancelCtx's children.
      removeChild(c.cancelCtx.Context, c)
   }
   c.mu.Lock()
   //将计时器重置
   if c.timer != nil {
      c.timer.Stop()
      c.timer = nil
   }
   c.mu.Unlock()
}
```



### valueCtx

首先valueCtx定义了一个结构体，其继承context接口

```go
type valueCtx struct {
   Context
   key, val interface{}
}
```

其定义了二个方法分别为String()、Value()。它们主要的作用是输出打印和输出key所对应的value。重点关注WithValue()函数。

```go
func WithValue(parent Context, key, val interface{}) Context {
   if key == nil {
      panic("nil key")
   }
   if !reflectlite.TypeOf(key).Comparable() {
      panic("key is not comparable")
   }
   return &valueCtx{parent, key, val}
}
```



### 总结

context包在平常运用的很多，尤其是在http方面居多，因此熟悉和了解其内部实现是很重要的事情。最后文章虽然写完了，但是总觉得还是有点懵，主要是关于父类和其子节点的Done()操作还有些细枝末节仍未摸清。