---
title: "go sync/atomic"
date: "2023-04-18 17:55:01"
categories:
  - Go
tags:
  - Go
toc: true
math: true
type: about
---

#### 定义
  原子操作是一种可以在不加锁的情况下，对内存数据进行读写的操作, 使用sync/atomic提供的原子操作可以确保在任意时刻只有一个goroutine对变量进行操作，避免并发冲突

#### 五种操作
  * 增减: AddXXX
  * 比较并交换: CompareAndSwapXXX
  * 载入: LoadXXX
  * 存储: StoreXXX
  * 交换: SwapXXX
  
  支持的数据类型: int32, int64, uint32, uint64, uintptr, unsafe.Pointer, 但是addXXX不支持unsafe.Pointer数据类型
  支持Value类型: CompareAndSwap, Load, Store, Swap方法

#### 使用

##### add
  ```
    // AddInt32 atomically adds delta to *addr and returns the new value.
    func AddInt32(addr *int32, delta int32) (new int32)

    // AddUint32 atomically adds delta to *addr and returns the new value.
    // To subtract a signed positive constant value c from x, do AddUint32(&x, ^uint32(c-1)).
    // In particular, to decrement x, do AddUint32(&x, ^uint32(0)).
    func AddUint32(addr *uint32, delta uint32) (new uint32)

    // AddInt64 atomically adds delta to *addr and returns the new value.
    func AddInt64(addr *int64, delta int64) (new int64)

    // AddUint64 atomically adds delta to *addr and returns the new value.
    // To subtract a signed positive constant value c from x, do AddUint64(&x, ^uint64(c-1)).
    // In particular, to decrement x, do AddUint64(&x, ^uint64(0)).
    func AddUint64(addr *uint64, delta uint64) (new uint64)

    // AddUintptr atomically adds delta to *addr and returns the new value.
    func AddUintptr(addr *uintptr, delta uintptr) (new uintptr)
  ```
  
  功能: 把 addr 指针 指向的内存里的值 和 delta做加法，然后返回新值
##### CompareAndSwap
  ```
    // CompareAndSwapInt32 executes the compare-and-swap operation for an int32 value.
    func CompareAndSwapInt32(addr *int32, old, new int32) (swapped bool)

    // CompareAndSwapInt64 executes the compare-and-swap operation for an int64 value.
    func CompareAndSwapInt64(addr *int64, old, new int64) (swapped bool)

    // CompareAndSwapUint32 executes the compare-and-swap operation for a uint32 value.
    func CompareAndSwapUint32(addr *uint32, old, new uint32) (swapped bool)

    // CompareAndSwapUint64 executes the compare-and-swap operation for a uint64 value.
    func CompareAndSwapUint64(addr *uint64, old, new uint64) (swapped bool)

    // CompareAndSwapUintptr executes the compare-and-swap operation for a uintptr value.
    func CompareAndSwapUintptr(addr *uintptr, old, new uintptr) (swapped bool)

    // CompareAndSwapPointer executes the compare-and-swap operation for a unsafe.Pointer value.
    func CompareAndSwapPointer(addr *unsafe.Pointer, old, new unsafe.Pointer) (swapped bool)
  ```
  
  功能: 比较 addr 指针指向的内存里的值是否为旧值old相等。如果相等，就把addr指针指向的内存里的值替换为新值new，并返回true。否则直接返回false
##### load
  ```
    // LoadInt32 atomically loads *addr.
    func LoadInt32(addr *int32) (val int32)

    // LoadInt64 atomically loads *addr.
    func LoadInt64(addr *int64) (val int64)

    // LoadUint32 atomically loads *addr.
    func LoadUint32(addr *uint32) (val uint32)

    // LoadUint64 atomically loads *addr.
    func LoadUint64(addr *uint64) (val uint64)

    // LoadUintptr atomically loads *addr.
    func LoadUintptr(addr *uintptr) (val uintptr)

    // LoadPointer atomically loads *addr.
    func LoadPointer(addr *unsafe.Pointer) (val unsafe.Pointer)
  ```

  功能: 返回 addr 指针指向的内存里的值
##### store
  ```
    // StoreInt32 atomically stores val into *addr.
    func StoreInt32(addr *int32, val int32)

    // StoreInt64 atomically stores val into *addr.
    func StoreInt64(addr *int64, val int64)

    // StoreUint32 atomically stores val into *addr.
    func StoreUint32(addr *uint32, val uint32)

    // StoreUint64 atomically stores val into *addr.
    func StoreUint64(addr *uint64, val uint64)

    // StoreUintptr atomically stores val into *addr.
    func StoreUintptr(addr *uintptr, val uintptr)

    // StorePointer atomically stores val into *addr.
    func StorePointer(addr *unsafe.Pointer, val unsafe.Pointer)
  ```

  功能: 把 addr 指针指向的内存里的值修改为 val
##### swap
  ```
    // SwapInt32 atomically stores new into *addr and returns the previous *addr value.
    func SwapInt32(addr *int32, new int32) (old int32)

    // SwapInt64 atomically stores new into *addr and returns the previous *addr value.
    func SwapInt64(addr *int64, new int64) (old int64)

    // SwapUint32 atomically stores new into *addr and returns the previous *addr value.
    func SwapUint32(addr *uint32, new uint32) (old uint32)

    // SwapUint64 atomically stores new into *addr and returns the previous *addr value.
    func SwapUint64(addr *uint64, new uint64) (old uint64)

    // SwapUintptr atomically stores new into *addr and returns the previous *addr value.
    func SwapUintptr(addr *uintptr, new uintptr) (old uintptr)

    // SwapPointer atomically stores new into *addr and returns the previous *addr value.
    func SwapPointer(addr *unsafe.Pointer, new unsafe.Pointer) (old unsafe.Pointer)
  ```

  功能: 把 addr 指针指向的内存里的值 替换为 新值new，然后返回 旧值old

#### 例子
  在日常中可能很少用到atomic，但还是得了解一下是如何在多个goroutine中操作使用的。经典的例子便是sync.Mutex

##### 例子1
  ```
    //多个goroutine对num2进行+1, 导致同一时间num2更新的值有误
    var num2 int64

    func main() {
        num2 = 0
        ch := make(chan string)
        for i := 0; i < 10000; i++ {
            go add2(ch, i)
        }
        for i := 0; i < 10000; i++ {
            <-ch
        }
        println(num2)
    }
    func add2(ch chan string, i int) {
        num2++
        ch <- ("add" + strconv.Itoa(i))
    }
  ```

  ```
    //使用atomic对num进行操作, 使得同一时间num只有一个goroutine有操作权
    var num int64

    func main() {
        num = 0
        ch := make(chan string)
        for i := 0; i < 10000; i++ {
            go add(ch, i)
        }
        for i := 0; i < 10000; i++ {
            <-ch
        }
        println(num)
    }
    func add(ch chan string, i int) {
        atomic.AddInt64(&num, 1)
        ch <- ("add" + strconv.Itoa(i))
    }
  ```