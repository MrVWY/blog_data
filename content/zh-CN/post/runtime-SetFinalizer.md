---
title: "go runtime.SetFinalizer 用法"
date: "2023-02-18 10:34:47"
categories:
  - Go
tags:
  - Go
toc: true
math: true
type: about
---

### 
  runtime.SetFinalizer可以在内存中对象被回收前(gc)，指定一些操作。  

### 用法
  document : https://pkg.go.dev/runtime#SetFinalizer
  ```
  //SetFinalizer sets the finalizer associated with obj to the provided finalizer function. When the garbage collector finds an unreachable block with an associated finalizer, it clears the   
  //association and runs finalizer(obj) in a separate goroutine. This makes obj reachable again, but now without an associated finalizer. Assuming that SetFinalizer is not called again, the next  
  //time the garbage collector sees that obj is unreachable, it will free obj.
  func SetFinalizer(obj any, finalizer any) 
  ``` 
  * 参数obj必须是指针类型
  * 参数finalizer是一个函数，其参数可有可无，类型是obj的类型，并且没有返回值

  当GC发现obj对象是unreachable(不可达)的时候，查看是否关联SetFinalizer函数  
   * 有则执行SetFinalizer函数，同时取消关联SetFinalizer函数。当下一次GC的时候，便可被回收  
   * 无则按正常GC流程进行回收

### 注意点
  1. SetFinalizer函数只能在GC触发的时候运行，因此即使程序正常结束或者发生panic，obj没有被GC选中清除，那么SetFinalizer也不会运行
  2. SetFinalizer函数按变量依赖顺序执行，如: A->B，只有A的SetFinalizer执行后，A被GC回收。B的SetFinalizer才能执行和被GC回收

### 例子
  1. [go-cache库](https://github.com/patrickmn/go-cache)
   ```
   type Cache struct {
      *cache
    }

    type cache struct {
      defaultExpiration time.Duration
      items             map[string]Item
      mu                sync.RWMutex
      onEvicted         func(string, interface{})
      janitor           *janitor
    }

    func New(defaultExpiration, cleanupInterval time.Duration) *Cache {
      items := make(map[string]Item)
      return newCacheWithJanitor(defaultExpiration, cleanupInterval, items)
    }

    func newCacheWithJanitor(de time.Duration, ci time.Duration, m map[string]Item) *Cache {
      c := newCache(de, m)
      C := &Cache{c}
      if ci > 0 {
        runJanitor(c, ci)
        runtime.SetFinalizer(C, stopJanitor)
      }
      return C
    }

    func runJanitor(c *cache, ci time.Duration) {
      j := &janitor{
        Interval: ci,
        stop:     make(chan bool),
      }
      c.janitor = j
      go j.Run(c)
    }

    func stopJanitor(c *Cache) {
      c.janitor.stop <- true
    }

    func (j *janitor) Run(c *cache) {
      ticker := time.NewTicker(j.Interval)
      for {
        select {
        case <-ticker.C:
          c.DeleteExpired()
        case <-j.stop:
          ticker.Stop()
          return
        }
      }
    }
   ```
  2. os.newFile
   ```
    // newFile is like NewFile, but if called from OpenFile or Pipe
    // (as passed in the kind parameter) it tries to add the file to
    // the runtime poller.
    func newFile(fd uintptr, name string, kind newFileKind) *File {
      ..........

      runtime.SetFinalizer(f.file, (*file).close) //防止file没有close
      return f
    }
   ```