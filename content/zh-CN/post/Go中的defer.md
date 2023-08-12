---
title: "Go中的defer"
date: "2020-06-04 01:42:26"
categories:
  - Go
tags:
  - Go
toc: true
math: true
type: about
---

例子1：

```go
func increaseA() int {
    var i int
    defer func() {
    i++
    }()
    return i
    //注意这是个匿名返回值
}

func increaseB() (r int) {
    defer func() {
    r++
    }()
    return r
}
```

例子2：

```go
func f1() (r int) {
    defer func() {
    r++
    }()
    return 0
}

func f2() (r int) {
    t := 5
    defer func() {
    t = t + 5
    }()
    return t
}

func f3() (r int) {
    defer func(r int) {
    r = r + 5
    }(r)
    return 1
}
```

例子3：

```go
func main() {
    defer func() {
    	fmt.Print(recover())
    }()
    defer func() {
    	defer func() {
    		fmt.Print(recover())
    	}()
    	panic(1)
    }()
    defer recover()
    panic(2)
}
```

跳坑关键：return ×× 这句话在编译过后会变为一下三条指令 1、返回值 = ×× 2、调用defer function 3、空的return

因此例子2的结果显而易见，分别为：1、5、1， 而例子3的结果为12，例子1的结果为0、1