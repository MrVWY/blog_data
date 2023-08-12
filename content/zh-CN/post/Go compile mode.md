---
title: "Go的几种编译模式"
date: "2020-08-28 22:21:23"
categories:
  - Go
tags:
  - Go
toc: true
math: true
type: about
---

Go与C家族的互调，Go的几种编译模式。
<!-- more -->
### 一个程序编译成可执行程需要经过哪些

会经过4个步骤：

1.  预编译：又称为预处理，是做些代码文本的替换工作。是整个编译过程的最先做的工作，生成**.i文件**。[参考](https://baike.baidu.com/item/%E9%A2%84%E7%BC%96%E8%AF%91)
2.  编译：主要对代码进行语法、语义等分析。检查无误后，翻译成二进制机器指令，生成汇编代码，将.i文件转化成**.s文件**。[参考](https://baike.baidu.com/item/%E7%BC%96%E8%AF%91)
3.  汇编：把上述的汇编代码翻译成二进制机器指令，将.s文件转化成**.o文件**。
4.  链接：又分为静态链接和动态链接，将众多的.o合成一个完整的可执行文件（静态/动态库文件）---Windows下为**.lib**和**.dll**文件(lib是编译时需要的，dll是运行时需要的。)，Linux下为**.a**和**.so**文件。

注：不同平台下，生成文件可能不同。例如window下，在汇编阶段会产生.obj文件。

### go 编译模式

关于go build 这个工具集是常用工具，能够将所写的代码打包编译成可执行文件(类型取决于所处平台)，也可以加上-buildmode指定编译模式。

#### \-buildmode参数

1.  \-buildmode=archive
    *   *   原版文档解释：This is the default build mode for a package that is not main.  It means to build the package into a .a file.  This is already supported for all targets.
        *   个人理解：这个参数主要用来将那么不是名为main的.go编译成.a文件，不过这种方式生成的.a文件。目前尚未弄清生成在哪个文件夹下。
2.  \-buildmode=default
    *   *   原版文档解释： Listed main packages are built into executables and listed non-main packages are built into .a files (the default behavior).
        *   个人理解：这个是默认情况下buildmode所采取的方法，其最终生成一个可执行文件和把除main.go外的其他.go编译成.a文件
3.  \-buildmode=shared：
    *   这个参数主要编译成一个.so文件、.a文件，不过要在命令后面配上-linkshared参数，可以参考一下这篇[文章](http://www.voidcn.com/article/p-mbxdfugq-brh.html)。
4.  \-buildmode=c-archive （ C动态链接文件）
    *   原版文档解释：Build the listed main package, plus all packages it imports, into a **C archive file**. The only callable symbols will be those functions exported using a cgo //export comment. Requires exactly one main package to be listed.
    *   注意：在使用go文件编译成能让C/C++引用的文件时，要主要必须要有的2个要求：第一是要在头部加入import "C"。第二是要在你想让C/C++引用的函数上方加入  //export  你的函数名。
    *   产生文件：.a 、.h。.h文件可以在C/C++引用![](Go compile mode/20200826095638331.png)
    *   参考链接-[stackoverflow](https://stackoverflow.com/questions/40573401/building-a-dll-with-go-1-7)
5.  \-buildmode=c-shared
    *   原版文档解释：Build the listed main package, plus all packages it imports, into a **C shared library**. The only callable symbols will be those functions exported using a cgo //export comment. Requires exactly one main package to be listed.
    *   大体上和c-archive模式差不多，主要是C archive file和C shared library的区别。按字面意思是file和library的区别。
    *   产生文件：一个文件(不知用来干啥)，.h文件供C/C++调用。![](Go compile mode/20200826095532942.png)

关于buildmode的参数还有exe、pie、plugin，这些参数就不展开多说。可以通过go help buildmode查看文档解析。

### C调Go、Go调C例子

#### C调Go

```
//A.go
import "C"
import "fmt"

//export A
func A() {
   fmt.Println("From DLL: A!")
}
```

func main() {}

通过**_go build -buildmode=c-archive -o ××.a  A.go_ ** 生成A.a 和 A.h文件。然后编写.c文件

```
#include "h.h"
#include <stdio.h>
int main()
{
    Bar();
    return 0;
}
```

编译C+运行：

```
gcc -o main hello.c h.a -undefined reference to \`pthread\_create'
```

 注：这里的-lpthread参数还需研究，就目前来看是能够调用成功。与其相关的还有一个-pthread参数，仍需自行查阅相关资料。 就目前来看-lpthread是与POSIX(可移植操作系统接口) thread相关 ![](Go compile mode/20200828140739493.png) 注意：上述只是个简单的例子，具体更复杂的问题涉及到C与Go直接的值范围等到一些问题还需要注意，因此在调用中更应该注重两种语言之间的类型转换。

#### Go调C

主要是运用了Go实现的CGO库调用，参考及内部实现：[链接](https://chai2010.cn/advanced-go-programming-book/ch2-cgo/ch2-05-internal.html) 加油！

### Reference

*   [静态链接(Static link)](https://baike.baidu.com/item/%E9%9D%99%E6%80%81%E9%93%BE%E6%8E%A5)
*   [动态链接(Dynamic Linking)](https://baike.baidu.com/item/%E5%8A%A8%E6%80%81%E9%93%BE%E6%8E%A5)
*    [Go Execution Modes](https://docs.google.com/document/d/1nr-TQHw_er6GOQRsF6T43GGhFDelrAP0NqSS_00RgZQ/view?pli=1#heading=h.fwmrrio0df0i)