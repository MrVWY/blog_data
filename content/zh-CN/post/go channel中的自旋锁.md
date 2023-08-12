---
title: "go channel中的自旋锁"
date: "2020-06-15 15:34:17"
categories:
  - Go
tags:
  - Go
toc: true
math: true
type: about
---

### 首先了解几个参数

```go
const (
   active_spin     = 4  //主动自旋
   active_spin_cnt = 30 
   passive_spin    = 1  //被动自旋
)
```

关于active\_spin\_cnt目前没有找到相关解释，后面找到会补上。

### hcan.lock

关键代码如下：

```go
func lock(l *mutex) {
   .......
   //  在单核CPU上没有自旋
   // 在多核CPU上存在自旋且不断的进行ACTIVE_SPIN的尝试
   spin := 0
   if ncpu > 1 {
      spin = active_spin
   }
Loop:
   for i := 0; ; i++ {
      v := atomic.Loaduintptr(&l.key)
      if v&locked == 0 {
         // 如果没有锁，尝试去锁上（抢占）
         if atomic.Casuintptr(&l.key, v, vlocked) {
            return
         }
         i = 0
      }
      if i < spin {
         procyield(active_spin_cnt) //主动自旋
      } else if i < spin+passive_spin {
         osyield()//被动自旋
      } else {
         //如果自旋之后还没获得，那么就插入到一个等待锁的队列中
         for {
            gp.m.nextwaitm = muintptr(v &^ locked)
            if atomic.Casuintptr(&l.key, v, uintptr(unsafe.Pointer(gp.m))locked) {
               break
            }
            v = atomic.Loaduintptr(&l.key)
            if v&locked == 0 {
               continue Loop
            }
         }
         if v&locked != 0 {
            // Queued. Wait.
            semasleep(-1)
            i = 0
         }
      }
   }
}
```



### 总结

关于channel的自旋锁大致就这些，无论它如果操作，最终都是有一段自旋的短暂时间。这个时间取决于是主动还是被动（i < spin 或者 i < spin+passive\_spin）。同时需要注意在channel中的lock字段是一个名为key的uinptr类型的值，在上锁或者解锁时，都是通过对其进行原子操作来改变其的值。因此由于这种机制，在这种情况下，goroutine就省去了从Grunning到Gopark再到Gready的过程。