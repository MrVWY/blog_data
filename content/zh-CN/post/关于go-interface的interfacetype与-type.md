---
title: "Go interface的interfacetype与_type"
date: "2020-08-06 00:37:35"
categories:
  - Go
tags:
  - Go
toc: true
math: true
type: about
---

### 题目一

```go
........
type T interface{}
type X string
type Y = string

func main() {
   var t T = "abc"
   var x X = "abc"
   var y Y = "abc"

   fmt.Println(t == x)
   fmt.Println(t == string(x))

   fmt.Println(t == y)
   fmt.Println(t == string(y))
}
// 输出false   true  true  true
```

### interface的内部结构

总所周知，在Go里面任何类型都实现了空接口，而非空接口则必须要实现其里面的方法才算是实现了这个非空接口。 当比较t==x时，t的动态类型是string，而x的动态类型是

#### 空接口结构

```go
type eface struct {
   _type *_type
   data  unsafe.Pointer
}
```

空接口的结构也十分简单，\_type是指向Go内部所定义的类型系统，在这就不展开述说，而data这指向接口的数据。

#### 非空接口结构

非空接口的构造相对复杂，这里就贴个本人画的图： ![](关于go-interface的interfacetype与-type/20200804170130760.png) 由此可见，在Go中的interface内部构造还是挺复杂的。

### 解题

解决题目的关键是要理解Go interface。

#### 前提

比如说请看下面一题

```go
var rw io.ReadWriter
v, _ := os.Open("test.txt")
rw = v
fmt.Printf("%T", rw)
//*os.File
```

问题来了，我们用一个var定义了一个io.ReadWriter类型的rw，之后一系列操作结果显示rw的类型为\*os.File 这里就涉及到iface结构里面的interfacetype与\_type这俩个字段了。interfacetype字段是指向**接口所定义的类型信息**，也可以理解为一个静态类型。而\_type字段则是**接口实际指向的类型信息**，也就是你人为的手动的赋值(也可称动态类型)。 由此这就可以解释清楚了，在Go里面所有类型都实现了空接口，而用var定义的rw初始的interfacetype指向了io.ReadWriter类型，之后通过os包打开了一个文件v，其类型为\*os.File。但又赋给rw，这时rw的\_type字段便指向了\*os.File类型。因此最后结果就为\*os.File。

#### 题目一的解法

首先分析一下三个type，type X string只是一个自定义类型，底层类型还是string，但X与string是不同类型的。而type Y = string，Y只是string的一个别名。 根据Go语言规范里面的一句话： 

```
A value `x` of non-interface type `X` and a value `t` of interface type `T` are comparable when values of type `X` are comparable and `X` implements `T`. They are equal if `t`'s dynamic type is identical to `X` and `t`'s dynamic value is equal to `x`.
```

 再结合上述所说的interface内部构造，可以得知t的动态类型(\_type)为string，动态值为abc，而x动态类型(\_type)为X，动态值为abc，因此 t != x 。

### Reference

*   [The Go Programming Language Specification(Go语言规范)](https://golang.org/ref/spec)
*   [题目来源](https://mp.weixin.qq.com/s/gSQBxGSTNwDfuliQuHppQg)