---
title: "CommonJS, AMD and RequireJS"
date: "2022-04-06 11:37:48"
categories:
  - Js
tags:
  - Js
toc: true
math: true
type: about
---

最近在使用typescript写一点小脚本，再用tsc编译成js文件后，使用node运行发现报错，再转去看一眼编译出来的js文件，发现不对劲。编译出来的js文件如下

```
define(["require", "exports", "./ProjectA/A", "./ProjectB/B"], function (require, exports, A_1, B_1) {
........
});
```

这时问题就来了，经过一番寻找发现编译出来的这种js文件和AMD有关。

### AMD

Asynchronous Module Definition，其思想上其实也就只有一种目的，那就是浏览器的按需加载，也就是异步加载模块，从而不影响后面的代码运行。[RequireJS](https://requirejs.org/docs/commonjs.html)就是其中实现AMD思想的一个js。

### CommonJs

commonjs最好的体现那就是nodejs的模块功能。其给我的感觉与AMD相比，最大的一点那就是commonjs的模块加载是同步处理的，这样就是阻塞后面的代码运行。

```
// someModule.js
exports.doSomething = function() { return "foo"; };

//otherModule.js
var someModule = require('someModule'); // in the vein of node    
exports.doSomethingElse = function() { return someModule.doSomething() + "bar"; };
```
