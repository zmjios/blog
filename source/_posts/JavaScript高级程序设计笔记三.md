---
title: JavaScript高级程序设计笔记三
date: 2015-04-01
desc: JavaScript高级程序,JavaScript,笔记
---

**函数表达式**

JS中定义函数有两种方式，一种是函数声明，另外一种是函数表达式。函数声明存在函数声明提升的过程，函数表达式则无此过程。

匿名函数，也叫lamda函数。即function后没有标识符

<!-- more -->


以下的代码在ECMAScript中属于无效语法：

```js
    if (true) {
      function say() {
        alert(“Hello”);
      }
    } else {
      function say() {
        alert(“No”);
      }
    }
```

应该使用匿名函数的方式来写：

```js
    var say;
    
    if (true) {
      say = function() {
        alert(“Hello”);
      }
    } else {
      say = function() {
    	alert(“No”);
      }
    }
```

JS中如果需要使用递归函数，那么应该使用arguments.callee来代替函数名，这样确保不会出现问题，最好不要直接填写自身函数名。