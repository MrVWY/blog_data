---
title: "go sync.Mutex"
date: "2020-06-17 21:58:43"
categories:
  - Go
tags:
  - Go
toc: true
math: true
type: about
---

### Mutex

![](/images/syncMap.png)

mutex有2种操作模式：正常和饥饿。在正常模式下，等待者按FIFO顺序排队，但唤醒的等待者不能马上获取锁，它还要和刚到达的goroutine们竞争，而且刚到的goroutine们有一个优势（它们已经在CPU上运行了）。因此刚唤醒的等待者可能很大几率会竞争失败。在这种情况下，它还会在这个队列中前面继续请求锁，如果请求时间超过1ms都没有就会将这个mutex对象（也就是锁）切换成饥饿模式。 在处于饥饿模式下，在队列最前面的等待者会直接获得这个锁的所有权，而其他刚到的goroutine们也不会尝试去获取锁的所有权，这时候也不会自旋。而是会加入到队列的末尾。 从饥饿模式切回到正常模式，触发条件为：（1）当前获得锁的所有权的等待者发现它是队列种的最后一个。（2）请求锁的时间小于1ms。符合这2种条件，那么当前获得锁的goroutine就会将锁切回正常模式。

#### 常量

```go
const (
   mutexLocked = 1 << iota // mutex is locked  
   mutexWoken //mutex唤醒
   mutexStarving //mutex饥饿
   mutexWaiterShift = iota //mutex转移
   starvationThresholdNs = 1e6 //mutex饥饿阈值
)
```

mutexLocked = 0001, mutexWoken = 0010, mutexStarving = 0100, mutexWaiterShift = 1000

#### 结构

```go
type Mutex struct {
   state int32 //当前Mutex的状态，0表示当前为unlocked状态
   sema  uint32 //信号量
}
```



#### 上锁

上锁这个过程分为fast path 和 slow path。 首先fast path过程十分简单，就是通过原子操作，对mutex.state检查和交换（如果mutex.state==0）

```go
func (m *Mutex) Lock() {
   // Fast path
   if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
      if race.Enabled {
         race.Acquire(unsafe.Pointer(m))
      }
      return
   }
   .....
}
```

接着如果fast path路径执行不了，说明该mutex.state!=0，于是进入了slow path过程，跟着代码走。

```go
func (m *Mutex) lockSlow() {
   var waitStartTime int64
   starving := false
   awoke := false //被唤醒标记，关于这个被唤醒标记，个人理解为，当自选到一定次数时，就会达到饥饿状态，那么我这个goroutine不能一直在这
   //进行自旋，因此到了饥饿状态，在下一次请求锁时，是一定可以获得锁的所有权的。因此在后面又要判断awoke将mutexWoken位置位0
   iter := 0 //自旋次数
   old := m.state
   for {

  //如果当前m.state = mutexLocked，通过runtime\_canSpin判断是否可以自旋
  //这里面的mutexLockedmutexStarving表示将mutexStarving位置位1，用于判断old是否为饥饿状态
  if old&(mutexLockedmutexStarving) == mutexLocked && runtime\_canSpin(iter) {

     //将m.state(old)置位为mutexWoken
   if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
        atomic.CompareAndSwapInt32(&m.state, old, oldmutexWoken) {
        awoke = true
    }
     runtime_doSpin() //自旋
     iter++
     old = m.state
     continue
  }

  new := old

  //如果old的mutexStarving位不为0，那么将new值的mutexLocker位置为1，表示不要取获取一个出于饥饿模式的锁，新来的goroutine要去到队列尾部
  if old&mutexStarving == 0 {
     new = mutexLocked
  }
  //如果old的mutexLocked和mutexStarving都不为0，那么说明当前已经上锁而且还处于饥饿状态，
  //将m.state的饥饿阈值+1
  if old&(mutexLockedmutexStarving) != 0 {
     new += 1 << mutexWaiterShift
  }
  //starving = true 且 old的mutexLocked位不为0，将new值的mutexStarving位置位1
  if starving && old&mutexLocked != 0 {
     new = mutexStarving
  }

  if awoke {
     //如果awoke为true，说明是被唤醒。
     if new&mutexWoken == 0 {
        throw("sync: inconsistent mutex state")
     }
     //重置mutexWoken位为0
     new &^= mutexWoken
  }
  //进行原子操作，可能要和其他刚到的goroutine们竞争
  if atomic.CompareAndSwapInt32(&m.state, old, new) {
     //如果old的mutexLocked位=0，说明抢锁成功
     if old&(mutexLockedmutexStarving) == 0 {
        break //正常模式抢锁成功，直接退出
     }
     //饥饿/正常模式抢锁
     queueLifo := waitStartTime != 0 第一次 queue 一定为false
     if waitStartTime == 0 {
        waitStartTime = runtime_nanotime() //获取等待时间
     }
     //如果queueLifo为true，将当前goroutine调到queue最前面
     runtime_SemacquireMutex(&m.sema, queueLifo, 1)
     starving = starving  runtime_nanotime()-waitStartTime > starvationThresholdNs //当前waitStartTime是否大于1ms
     old = m.state

     if old&mutexStarving != 0 {
        //如果是饥饿模式
        if old&(mutexLockedmutexWoken) != 0  old>>mutexWaiterShift == 0 {
           throw("sync: inconsistent mutex state")
        }
        delta := int32(mutexLocked - 1<<mutexWaiterShift)

        //饥饿模式变回正常模式
        if !starving  old>>mutexWaiterShift == 1 {
           delta -= mutexStarving
        }
        atomic.AddInt32(&m.state, delta)//更新，完成
        break
     }
     awoke = true //不是饥饿模式，又抢锁失败，重新发起抢锁竞争
     iter = 0
  } else {
     old = m.state //重新发起竞争
  }

   }
}
```

上锁过程尤其要注意slow path过程，首先它先判断当前模式是否为饥饿模式，如果是饥饿模式不让自旋。接着自旋后面是判断计算new的值分3种情况。接着就进行CAS原子操作，也分几种情况，一种是正常/饥饿模式抢锁失败，另一种是正常/饥饿模式抢锁成功，具体流程结合上面代码理解。

#### 解锁

解锁过程相对于上锁过程比较简洁

```go
func (m *Mutex) Unlock() {

   // 将m.state中的mutexLocked位置位0
   new := atomic.AddInt32(&m.state, -mutexLocked)
   if new != 0 {
      m.unlockSlow(new)
   }
}

func (m *Mutex) unlockSlow(new int32) {
   if (new+mutexLocked)&mutexLocked == 0 {
      throw("sync: unlock of unlocked mutex")
   }
   //查看new中的mutexStarving是否位0
   if new&mutexStarving == 0 {
      old := new
      for {
         //如果没有goroutine在等待锁，那么直接return
         if old>>mutexWaiterShift == 0  old&(mutexLockedmutexWokenmutexStarving) != 0 {
            return
         }
         new = (old - 1<<mutexWaiterShift)  mutexWoken
         if atomic.CompareAndSwapInt32(&m.state, old, new) {
            runtime_Semrelease(&m.sema, false, 1)
            return
         }
         old = m.state
      }
   } else {
      //释放信号量唤醒在等待的goroutine
      runtime_Semrelease(&m.sema, true, 1)
   }
}
```