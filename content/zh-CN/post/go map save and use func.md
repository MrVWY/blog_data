---
title: "go map 存储方法"
date: "2022-04-16 11:51:57"
categories:
  - Go
tags:
  - Go
toc: true
math: true
type: about
---

#### 背景

最近在公司重构个系统的时候，由于某一个流程需要按需来自定义功能来进行处理，于是我想着用一个map来做一个映射关系，然后在这个流程中利用一个标记来匹配map中的自定义功能函数，以此达到按需处理。但是一番敲击之后，发现效果不尽人意，最主要的还是如果要用map做映射关系，那么就必须要在开头import一堆自定义函数包。

例如下面这个例子

```
// server/
package main

import (
	"fmt"
    "server/a"
	"server/b"
	.......
)

func main() {

    functions := buildFunctions()
    f := functions["isInValid"]

    //  f("hello")

}

func buildFunctions() map[string]func() bool {

    functions := map[string]func() bool{
        "isInValid":   a.IsInValid,
        "isAvailable": b.IsAvailable,
        ......
    }
    return functions
}

// server/a/
func IsInValid(s string) bool {
    fmt.Println("Invalid ", s)
    return true
}
// server/B/
func IsAvailable(s string, s1 string) bool {
    return true
}
```

#### 思考

由于旧系统是用nodejs写的，其可以动态的将函数require存进map中，我们只需要定义好路径和函数名字，就可以以拼接的方式将函数require存进map中，如下所示

```

entry[mod] = load_module mod for mod in [
  "a"
  "b"
  ...... //函数名
]

module.exports = load_module = (module_name, proj = argv.project) ->
  try
    obj_common = require "./#{module_name}"
  catch err
    debug "未找到 #{module_name}, #{err.stack}".bold.red
  try
    obj_proj = require "./proj/#{proj}/#{module_name}"
    # debug "./proj/#{proj}/#{module_name}"
  catch err
    # debug "#{err}"
  return _.assign obj_common, obj_proj if "function" != typeof obj_common
  obj_proj or obj_common

//other js file
module.exports = a = (str) ->
  console.log("a", str)
  
//other js file
module.exports = b = (str) ->
  console.log("b", str)

//最终就可以生成一个map对象
entry[a] = a
entry[b] = b
```

基于旧系统的这种处理方式，在go中利用map去一个一个在map中做映射就会导致开头会出现一大堆import包的语句。

#### 解决

在与公司的大佬的建议下，他建议使用typescript编写这一功能，然后转成lua代码。最后嵌入进golang中。于是本人变浅尝一下。

```
//main.ts
let A = new Map()
function L(s:string) {
    let a = require("./ProjectA/" + s)
    A.set(s,a.My)
}

function Say(s:string) {
    L(s)
    let a = A.get(s)
    console.log("s", s)
    console.log("a", a)
    a()
}

//ProjectA/A
export function My() {
    console.log("This is a function A ! ") 
}

//tsconfig.json
{
  "compilerOptions": {
    "target": "esnext",
    "lib": ["esnext","dom"],
    "moduleResolution": "node",
    "types": ["node"],
    "strict": true
  },
  "tstl": {
    "luaTarget": "5.3",
    "noImplicitSelf":true,
    // 是否打包为单个lua文件
    "luaBundle":"api.lua",
    "luaBundleEntry":"main.ts"
  }
}

cmd : npm run build
```

通过上述运行npm run build命令之后会得到一个api.lua文件，接着我们把它嵌入进go中，使其通过传一个参数A去加载ProjectA/A中的My函数。

```
package main

import (
	"fmt"
	lua "github.com/yuin/gopher-lua"
)

func main() {
	L := lua.NewState()
	defer L.Close()
	if err := L.DoFile("api.lua"); err != nil {
		panic(err)
	}

	if err := L.CallByParam(lua.P{
		Fn:      L.GetGlobal("Say"),
		NRet:    1,    // 指定返回值数量
		Protect: true, // 如果出现异常，是panic还是返回err
	}, lua.LString("A")); err != nil {
		fmt.Println(err.Error())
	}

}

//out
s       A                     
a       function: 0xc000219b80
This is a function A !   
```

#### 总结

通过这种方式，我们可以很灵活的在golang去动态的加载某个自定义的方法和功能

#### Reference

1. https://stackoverflow.com/questions/52298469/map-to-store-generic-type-functions-in-go
2. https://github.com/yuin/gopher-lua
3. https://typescripttolua.github.io/
4. https://gitee.com/zmwcodediy/lua-ts-demo
