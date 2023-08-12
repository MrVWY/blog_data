---
title: "go sync.Map"
date: "2020-06-10 11:32:23"
categories:
  - Go
tags:
  - Go
toc: true
math: true
type: about
---

### sync.Map的主要结构

#### Map

```go
type Map struct {
   mu Mutex  //锁
   read atomic.Value // readOnly 只读数据 
   dirty map[interface{}]*entry  //存储数据，允许读写，但要用锁
   //未命中次数，主要记录read读取不到数据，加锁去读取dirty中数据的次数
   //到达一定次数时，dirty中的数据就会复制到read中，并且会重新再创建一个dirty
   misses int
}
```



#### readOnly

```go
// readOnly是用于存储，通过原子操作在map中的read字段
type readOnly struct {
   m       map[interface{}]*entry
   // 如果为true则表示some key在dirty存在，而readOnly这没有
   amended bool 
}
```



#### entry

```go
type entry struct {
   // 如果p==nil，表示已经被删除，并且m.dirty=nil
   // 如果p==expunged，表示已经被删除，但是该键已经不在dirty map中
   // 其他情况，表示在m.read和m.dirty都存有数据
   p unsafe.Pointer // \*interface{}
}
```



### Map结构主要函数

#### Load()

该函数首先会从read map 中取出数据，如果read中没有所需要的数据就去dirty map 中读取数据

```go
func (m *Map) Load(key interface{}) (value interface{}, ok bool) {
   read, _ := m.read.Load().(readOnly)
   e, ok := read.m[key]
   if !ok && read.amended {
      m.mu.Lock()
      //防止第一次从read取数据时碰上数据正从dirty复制到read中
      //因此这里还需再读取一次
      read, _ = m.read.Load().(readOnly)
      e, ok = read.m[key]
      if !ok && read.amended {
         e, ok = m.dirty[key]
         //未命中次数+1
         m.missLocked()
      }
      m.mu.Unlock()
   }
   if !ok {
      return nil, false
   }
   return e.load()//调取entry类所定义的load方法
}
```



#### store()

该函数主要用于存储数据。具体流程是： 1、先从read map中看看能找到key/value，如果找到通过e.tryStore函数更新数据 2、当从read map中没有找到数据，上锁，对dirty map进行操作。首先先check一下read map状态，如果在read map中找到，但仍是expunged状态，就通过e.unexpungeLocked()将expunged标记为nil，最后再把数据存储到entry中。 3、如果在read map 和dirty map 都没有找到数据，那么就判断read.amended是否为false，如果为false那么就说明在dirty map中没有该数据，因此就通过m.dirtyLocked去新建dirty map。

```go
func (m *Map) Store(key, value interface{}) {
   read, _ := m.read.Load().(readOnly)
   if e, ok := read.m[key]; ok && e.tryStore(&value) {
      return
   }

   m.mu.Lock()
   read, _ = m.read.Load().(readOnly)
   if e, ok := read.m[key]; ok {
      if e.unexpungeLocked() {
         m.dirty[key] = e
      }
      e.storeLocked(&value)
   } else if e, ok := m.dirty[key]; ok {
      e.storeLocked(&value)
   } else {
      if !read.amended {
         m.dirtyLocked()
         m.read.Store(readOnly{m: read.m, amended: true})
      }
      m.dirty[key] = newEntry(value)
   }
   m.mu.Unlock()
}
```

另外在源码中，有一处地方需要我们关注一下，请看上面加粗代码，通过观察我们可以发现从read.m\[key\]取出来的e，居然赋值给了m.dirty\[key\]，由此可以推测在底层中read map 和dirty map使用的是不同的指针，但是它们指向的都是同一个值。

#### Delete()

这个函数的代码流程就十分清晰了，首先它是先check一下read map中是否含有该数据如果没有而且read.amended为true，那么就上锁去dirty map中删除该数据，如果在read map中找到就直接删除即可。

```go
func (m *Map) Delete(key interface{}) {
   read, _ := m.read.Load().(readOnly)
   e, ok := read.m[key]
   if !ok && read.amended {
      m.mu.Lock()
      read, _ = m.read.Load().(readOnly)
      e, ok = read.m[key]
      if !ok && read.amended {
         delete(m.dirty, key)
      }
      m.mu.Unlock()
   }
   if ok {
      e.delete()
   }
}
```



#### Range()

这个主要用于遍历，通过检查read.amended是否为true，然后去dirty map或者read map遍历。

#### 总结

sync.map原理也不难，主要思想是空间换时间的思路，使用两个map来进行操作实现对锁的操作，一个read map主要用于并发中的读取和**已经存在的写(请看store函数中加粗的部分)**，一个dirty map主要用于读写。但有几个关键的状态还需注意一下分别是read.amended和expunded，其中read.amended状态量决定了是对哪个map。