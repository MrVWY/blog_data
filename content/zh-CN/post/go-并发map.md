---
title: "go 如何实现线程安全（并发安全）的 map"
date: "2022-05-27 11:02:08"
categories:
  - Go
tags:
  - Go
toc: true
math: true
type: about
---

在网上查了查如何实现线程安全（并发安全）的 map，发现了几种方法。

### sync.RWMutex

加读写锁

```go
//例子
type RWMap struct { // 一个读写锁保护的线程安全的map
    sync.RWMutex // 读写锁保护下面的map字段
    m map[int]int
}
// 新建一个RWMap
func NewRWMap(n int) *RWMap {
    return &RWMap{
        m: make(map[int]int, n),
    }
}
func (m *RWMap) Get(k int) (int, bool) { //从map中读取一个值
    m.RLock()
    defer m.RUnlock()
    v, existed := m.m[k] // 在锁的保护下从map中读取
    return v, existed
}

func (m *RWMap) Set(k int, v int) { // 设置一个键值对
    m.Lock()              // 锁保护
    defer m.Unlock()
    m.m[k] = v
}

func (m *RWMap) Delete(k int) { //删除一个键
    m.Lock()                   // 锁保护
    defer m.Unlock()
    delete(m.m, k)
}

func (m *RWMap) Len() int { // map的长度
    m.RLock()   // 锁保护
    defer m.RUnlock()
    return len(m.m)
}

func (m *RWMap) Each(f func(k, v int) bool) { // 遍历map
    m.RLock()             //遍历期间一直持有读锁
    defer m.RUnlock()

    for k, v := range m.m {
        if !f(k, v) {
            return
        }
    }
}
```

​	功能上满足，但是在高并发的场景下，性能跟不上，一旦加上锁，就必须等到锁释放其他协程才能使用

### 分片加锁

​	文档里说`concurrent-map`提供了一种高性能的解决方案:通过对内部`map`进行分片，降低锁粒度，从而达到最少的锁等待时间(锁冲突)。意思就是将 map 分成 n 块，每个块之间的读写操作都互不干扰，从而降低冲突的可能性。

​	直接看源码：https://github.com/orcaman/concurrent-map/blob/master/concurrent_map.go

​	Github：https://github.com/orcaman/concurrent-map

<img src="go-并发map/1.jpg"  />

​	其实就是定义`ConcurrentMapShared`结构体（这便是map里面的某个块），里面含有一个读写锁。

<img src="go-并发map/2.jpg" style="zoom:150%;" />

![](go-并发map/3.jpg)

​	当对map进行相关操作时，就去计算hash值，算出落在哪一个块中，再去进行相关操作。

### sync.map 

`sync.Map`在读多写少性能（读取 read 并不需要加锁，而读或写 dirty 都需要加锁）比较好，否则并发性能很差。关于sync.map的理解，在网上找了几张理解图，如下

第一张是sync.map的结构图

![](go-并发map/5.png)

第二张是关于sync.map里面的dirty和read的转换图

![](go-并发map/4.jpg)

关于从dirty迁移到read，当misses达到dirty长度时，才触发迁移，迁移完成之后misses置为0，dirty置为nil，read就拥有了dirty原来的数据。

总结：

- 通过 read 和 dirty 两个字段将读写分离，读的数据存在只读字段 read 上，将最新写入的数据则存在 dirty 字段上
- 读取时会先查询 read，不存在再查询 dirty，写入时则只写入 dirty
- 读取 read 并不需要加锁，而读或写 dirty 都需要加锁
- 另外有 misses 字段来统计 read 被穿透的次数（被穿透指需要读 dirty 的情况），超过一定次数则将 dirty 数据同步到 read 上
- 对于删除数据则直接通过标记来延迟删除

源码：https://github.com/golang/go/blob/master/src/sync/map.go
