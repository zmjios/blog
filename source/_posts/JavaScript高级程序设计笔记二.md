---
title: JavaScript高级程序设计笔记二
date: 2015-03-30 11:45:31
desc: JavaScript高级程序,JavaScript,笔记
---

**引用类型**
ECMAScript是面向对象的语言，但是并没用类的概念，因为没有类和接口。所以我们一般称之为引用类型。

<!-- more -->

以下是创建Array类型的几种方式：

```js
    var colors = new Array();
    var colors = new Array(3);
    var colors = new Array("blue", "red", "green");
    var colors = Array(3);

    var colors = ["red", "blue", "green"];
    var values = [1, 2,]; //这样会创建2或3项的数组（IE8或之前创建3个元素）
```

数组和对象使用字面量来定义时，不会调用构造函数

数组对象如果超过了索引，它会自动增加长度

数组中的length属性不是只读的，是可写的，可以通过修改length的值来移除末尾的项

alert一个数组，会后台自动调用toString()方法，与其效果一致

数组中null或undefined返回字符串时将以空字符串来表示

数组中的栈方法使用`push()` `pop()` 队列方法使用`push()` `shift()` `unshift()`是在第一个位置插入一直数值并返回数组长度

数组中`sort()`会调用每个数组项的`toString()`方法，然后再按照升序来排序

可以利用如下方法来对数字数组进行排序

```js
    values.sort(function(value1, value2){
       return value1 - value2;
    });
```

concat()可以复制一个数组，还可以传入新的参数，放在元数组的末尾。`slice(1)`从位置1开始复制到末尾，`slice(1, 4)`，从1赋值到位置3结束。
`splice(0,2, "red", "green");`  替换或者删除数组中的元素并返回一个被删除的元素的数组

函数是对象，函数名是指针。定义函数的三种方式:

```js

    function sum(val1,val2){
        return val1+val2;
    }
    
    var sum = function(va1, val2){
        return val1+val2;
    };   //这里有个分号，跟定义变量一致

    //不推荐这样的用法，性能比较低，解析了两次代码
    var sum = new Function("val1", "val2", "return val1+val2");
```

代码执行之前，解析器会执行 函数声明提升的过程，也就是说，函数的声明可以在调用之后。JS引擎会自动在执行前把声明提前。

如果通过等价来声明函数表达式，并在之前调用。则会出现错误。

`arguments`对象有一个calee的属性，该属性是一个指针，指向拥有这个arguments对象的函数

函数有两个属性，*length*和*prototype*。length表示函数希望接受到的参数个数

call(作用域, 每一次参数 )  apply(作用域，参数数组)  `bind(作用域)` 这三个函数都可以改变函数中this所指代的作用域

基本数据类型不是对象，但是会在后台执行基本包装类型的过程，这个与引用类型存在生存期的区别。