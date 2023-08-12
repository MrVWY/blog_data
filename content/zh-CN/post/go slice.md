---
title: "go slice"
date: "2022-07-06 21:11:34"
categories:
  - Go
tags:
  - Go
toc: true
math: true
type: about
---

### 1、切片结构

一个切片由3部分组成：**指针**、**长度**和**容量**。指针指向底层数组，长度代表slice当前的长度，容量代表底层数组的长度，同时切片的长度可以在运行时修改，最小为 0 最大为相关数组的长度：切片是一个长度可变的数组。

```go
type slice struct {
   array unsafe.Pointer
   len   int
   cap   int
}

s = [2]int{1, 2} //数组
s = []int{1, 2} //切片
```

### 2、nil和空切片

nil 切片：描述一个不存在的切片（无指针地址）

空切片：一个空的集合

区别：空切片指向的地址不是nil，指向的是一个内存地址，但是它没有分配任何内存空间，即底层元素包含0个元素

### 3、切片扩容

扩容代码

```go
func growslice(et *_type, old slice, cap int) slice {
   if raceenabled {
      callerpc := getcallerpc()
      racereadrangepc(old.array, uintptr(old.len*int(et.size)), callerpc, funcPC(growslice))
   }
   if msanenabled {
      msanread(old.array, uintptr(old.len*int(et.size)))
   }

   if cap < old.cap {
      panic(errorString("growslice: cap out of range"))
   }

   if et.size == 0 {
      // append should not create a slice with nil pointer but non-zero len.
      // We assume that append doesn't need to preserve old.array in this case.
      return slice{unsafe.Pointer(&zerobase), old.len, cap}
   }
   
   //扩容
   newcap := old.cap
   doublecap := newcap + newcap
   if cap > doublecap {
      newcap = cap
   } else {
      if old.cap < 1024 {
         newcap = doublecap
      } else {
         // Check 0 < newcap to detect overflow
         // and prevent an infinite loop.
         for 0 < newcap && newcap < cap {
            newcap += newcap / 4
         }
         // Set newcap to the requested cap when
         // the newcap calculation overflowed.
         if newcap <= 0 {
            newcap = cap
         }
      }
   }

// 计算新的切片的容量、长度
   var overflow bool
   var lenmem, newlenmem, capmem uintptr
   // Specialize for common values of et.size.
   // For 1 we don't need any division/multiplication.
   // For sys.PtrSize, compiler will optimize division/multiplication into a shift by a constant.
   // For powers of 2, use a variable shift.
   switch {
   case et.size == 1:
      lenmem = uintptr(old.len)
      newlenmem = uintptr(cap)
      capmem = roundupsize(uintptr(newcap))
      overflow = uintptr(newcap) > maxAlloc
      newcap = int(capmem)
   case et.size == sys.PtrSize:
      lenmem = uintptr(old.len) * sys.PtrSize
      newlenmem = uintptr(cap) * sys.PtrSize
      capmem = roundupsize(uintptr(newcap) * sys.PtrSize)
      overflow = uintptr(newcap) > maxAlloc/sys.PtrSize
      newcap = int(capmem / sys.PtrSize)
   case isPowerOfTwo(et.size):
      var shift uintptr
      if sys.PtrSize == 8 {
         // Mask shift for better code generation.
         shift = uintptr(sys.Ctz64(uint64(et.size))) & 63
      } else {
         shift = uintptr(sys.Ctz32(uint32(et.size))) & 31
      }
      lenmem = uintptr(old.len) << shift
      newlenmem = uintptr(cap) << shift
      capmem = roundupsize(uintptr(newcap) << shift)
      overflow = uintptr(newcap) > (maxAlloc >> shift)
      newcap = int(capmem >> shift)
   default:
      lenmem = uintptr(old.len) * et.size
      newlenmem = uintptr(cap) * et.size
      capmem, overflow = math.MulUintptr(et.size, uintptr(newcap))
      capmem = roundupsize(capmem)
      newcap = int(capmem / et.size)
   }

   if overflow || capmem > maxAlloc {
      panic(errorString("growslice: cap out of range"))
   }

   
   //确定 array unsafe.Pointer 地址
   var p unsafe.Pointer
   if et.ptrdata == 0 {
      p = mallocgc(capmem, nil, false)
      // The append() that calls growslice is going to overwrite from old.len to cap (which will be the new length).
      // Only clear the part that will not be overwritten.
      memclrNoHeapPointers(add(p, newlenmem), capmem-newlenmem)
   } else {
      // Note: can't use rawmem (which avoids zeroing of memory), because then GC can scan uninitialized memory.
      p = mallocgc(capmem, et, true)
      if lenmem > 0 && writeBarrier.enabled {
         // Only shade the pointers in old.array since we know the destination slice p
         // only contains nil pointers because it has been cleared during alloc.
         bulkBarrierPreWriteSrcOnly(uintptr(p), uintptr(old.array), lenmem-et.size+et.ptrdata)
      }
   }
    
   //迁移数据
   memmove(p, old.array, lenmem)

   return slice{p, old.len, newcap}
}
```

#### 切片扩容的策略：

```go
newcap := old.cap
doublecap := newcap + newcap //2倍
if cap > doublecap {
   newcap = cap
} else {
   if old.cap < 1024 {
      newcap = doublecap
   } else {
      // Check 0 < newcap to detect overflow
      // and prevent an infinite loop.
      for 0 < newcap && newcap < cap {
         // 1/4
         newcap += newcap / 4
      }
      // Set newcap to the requested cap when
      // the newcap calculation overflowed.
      if newcap <= 0 {
         newcap = cap
      }
   }
}
```

- 首先判断，如果新申请容量（cap）大于2倍的旧容量（old.cap），最终容量（newcap）就是新申请的容量（cap）
- 否则判断，如果旧切片的长度小于1024，则最终容量(newcap)就是旧容量(old.cap)的两倍，即（newcap=doublecap）
- 否则判断，如果旧切片长度大于等于1024，则最终容量（newcap）从旧容量（old.cap）开始循环增加原来的 1/4，直到最终容量（newcap）大于等于新申请的容量(cap)，即（newcap >= cap）
- 如果最终容量（cap）计算值溢出，则最终容量（cap）就是新申请容量（cap）

### 4、切片扩容的两种情况

#### 情况一

```go
func main() {

   array := [5]int{1, 2, 3, 4, 5} //数组
   slice := array[0:2]            //切片  len=2 cap=5 array = &array
   newSlice := append(slice, 100)
   fmt.Printf("Before slice = %v, Pointer = %p, len = %d, cap = %d\n", slice, &slice, len(slice), cap(slice))
   fmt.Printf("Before newSlice = %v, Pointer = %p, len = %d, cap = %d\n", newSlice, &newSlice, len(newSlice), cap(newSlice))
   newSlice[1] += 10
   fmt.Printf("After slice = %v, Pointer = %p, len = %d, cap = %d\n", slice, &slice, len(slice), cap(slice))
   fmt.Printf("After newSlice = %v, Pointer = %p, len = %d, cap = %d\n", newSlice, &newSlice, len(newSlice), cap(newSlice))
   fmt.Printf("After array = %v\n", array)

}

//打印输出
Before slice = [1 2], Pointer = 0xc000004078, len = 2, cap = 5
Before newSlice = [1 2 100], Pointer = 0xc000004090, len = 3, cap = 5
After slice = [1 12], Pointer = 0xc000004078, len = 2, cap = 5       
After newSlice = [1 12 100], Pointer = 0xc000004090, len = 3, cap = 5
After array = [1 12 100 4 5]
```

结合切片的结构可以看出，由于原数组还有容量cap可以使用，所以执行 append() 操作以后，会在原数组上直接操作，所以这种情况下，扩容以后的数组还是指向原来的数组。

因此多个slice指向相同的底层数组时，修改其中一个slice，可能会影响其他slice的值。

#### 情况二

```go
var p unsafe.Pointer
if et.ptrdata == 0 {
   p = mallocgc(capmem, nil, false)
   // The append() that calls growslice is going to overwrite from old.len to cap (which will be the new length).
   // Only clear the part that will not be overwritten.
   memclrNoHeapPointers(add(p, newlenmem), capmem-newlenmem)
} else {
   // Note: can't use rawmem (which avoids zeroing of memory), because then GC can scan uninitialized memory.
   //重新申请capmem大小的内存地址，并初始化为zero
   p = mallocgc(capmem, et, true)
   if lenmem > 0 && writeBarrier.enabled {
      // Only shade the pointers in old.array since we know the destination slice p
      // only contains nil pointers because it has been cleared during alloc.
      //执行写屏障，将p给打上颜色
      bulkBarrierPreWriteSrcOnly(uintptr(p), uintptr(old.array), lenmem-et.size+et.ptrdata)
   }
}
//迁移数据
memmove(p, old.array, lenmem)
```

数组的容量已经达到了最大值，Go 默认会先开一片新的内存区域，把原来的值拷贝过来，然后再执行 append() 操作。这时候新旧切片所指向的数组的内存地址不一样，因此在新切片上操作不会对旧切片造成影响。

### 5、切片copy

```go
// slicecopy is used to copy from a string or slice of pointerless elements into a slice.
func slicecopy(toPtr unsafe.Pointer, toLen int, fromPtr unsafe.Pointer, fromLen int, width uintptr) int {
   if fromLen == 0 || toLen == 0 {
      return 0
   }

   n := fromLen
   if toLen < n {
      n = toLen
   }

   if width == 0 {
      return n
   }

   size := uintptr(n) * width
   if raceenabled {
      callerpc := getcallerpc()
      pc := funcPC(slicecopy)
      racereadrangepc(fromPtr, size, callerpc, pc)
      racewriterangepc(toPtr, size, callerpc, pc)
   }
   if msanenabled {
      msanread(fromPtr, size)
      msanwrite(toPtr, size)
   }

   if size == 1 { // common case worth about 2x to do here
      // TODO: is this still worth it with new memmove impl?
      *(*byte)(toPtr) = *(*byte)(fromPtr) // known to be a byte pointer
   } else {
      memmove(toPtr, fromPtr, size)
   }
   return n
}
```

例

```go
func main() {

   array := []int{10, 20, 30, 40}
   slice := make([]int, 6)
   n := copy(slice, array)
   fmt.Println(n, slice)

}

//output
4 [10 20 30 40 0 0]
```

注意

```go
func main() {

   slice := []int{10, 20, 30, 40}
   for index, value := range slice {
      fmt.Printf("value = %d , value-addr = %x , slice-addr = %x\n", value, &value, &slice[index])
   }

}

//output
value = 10 , value-addr = c000018098 , slice-addr = c00000e200
value = 20 , value-addr = c000018098 , slice-addr = c00000e208
value = 30 , value-addr = c000018098 , slice-addr = c00000e210
value = 40 , value-addr = c000018098 , slice-addr = c00000e218
```

说明value的内存地址上的值的从slice复制过来的，直接操作value是不会对slice本身造成什么影响的。
