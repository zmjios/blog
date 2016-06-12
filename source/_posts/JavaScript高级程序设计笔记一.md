---
title: JavaScript高级程序设计笔记一
date: 2015-03-29 11:45:31
desc: JavaScript高级程序,JavaScript,笔记
---

阅读JavsScript高级程序设计（第三版）中记录一些个人感觉一些容易被忽视的知识点。

<!-- more -->

JavaScript中可以使用字母，下划线以及$符号作为开头。

JS中可以省略分号，但是不推荐。使用分号有三个好处:

1. 避免一些错误;
2. 增进解析的速度;
3. 可以通过删除空格来压缩JS代码。

JS函数中使用var声明的是局部变量，缺省var则是全局变量。但是不推荐使用缺省var的方式来声明全局变量，容易造成混乱。

Undefined和Null类型是只有一个值的数据类型分别为 undefined和null。

若强制输出一个未定义的变量，则会产生错误。

```js
var message;
alert(message);  //"undefined"
alert(age);      //Error
```

对未声明的变量执行typeof操作符，也返回undefined值

```js
var age;
alert(age);    //"undefined"
alert(mess);   //"undefined"
```

```js    
alert(typeof null)   // "object"  原因null表示空对象指针。
undefined 派生于 null。所以
alert(null == undefined);  // true
```

Boolean类型中的字面量true、false与数字无关，并非1和0。

ECMAScript会不失时机的将伪浮点数值转换成浮点类型累储存，从而节省一半的储存空间。

```js
var fNum = 1.;     //为1
var fNum2 = 10.0;  //为10
```

JS中浮点数值最高精度为17位小数，该类型采用了IEEE754格式，IEEE754浮点计算有一个通病，即浮点的计算精确度存在一些问题。例如:

```js
var a=0.1;
var b=0. 2;
alert(a+b);   //0.30000000000000004
```

> 因此不要做这样的判断 if(a+b == 0.3){}

数值的范围为 Number.MIN_VALUE - Number.MAX_VALUE 超出范围则变成 (-Infinity[Number.NEGATIVE_INFINITY] || Infinity[Number.POSITIVE_INFINITY]) 不能进行计算。

ECMAScript中 任何数除以0 等到的NaN 并不会报错

NaN 与任何数都不相等，包括本身。 NaN != NaN

isNaN()会对传入的类型先进行转换成数字之后再进行判断。

```js
isNaN(null)  // false
Number("") //0
Number("HelloJs") //NaN
Number(null)  //0
Number(undefined) //NaN
=>一元加操作符与Number()一致
```
ECMAScript中，用单引号和双引号引用字符串效果一致。但是不可以混用。
    var lang = "JAVA" 
    lang = lang + "Script";

> 过程是创建一个长度为10的新字符串，然后填充JAVA和SCRIPT。最后删除两个没用的字符串，保留 lang = "JAVAScript"

如果不需要传参的话，ECMAScript中可以省略var obj = new Object()的括号。
但是不推荐这样的用法。

逻辑与是短路操作，即第一个操作数能够决定结果就不会再计算第二个操作数。逻辑或与此类似

```js
    var res = (true && undefinedVariable);    //Error，中断程序
    alert(res);   //不会被执行
        
    var res = (false && undefinedVariable)  //false
    alert(res)  //false
    var myObj = obj || backupObj;
```

这种赋值模式，如果第一个变量不为null，返回第一个对象，否则返回第二个对象。

加法中，如果只有一个操作数是字符串，那么将另一个操作数转换成字符串之后，两个字符串进行拼接.

```js
var a = 5;
var b = "5";
alert(a+b);  //"55"
```
        
如果有三个操作数，其中一个是字符串，则类似：

```js
var num1 = 5;
var num2 = 10;
var mess = "The sum of 5 and 10 is " +  num1 + num2;  // ...+(num1 + num2); 这样才是得到15
alert(mess); // The sum of 5 and 10 is 510
```
        
valueOf偏向于运算，toString偏向于显示。

1. 在进行对象转换时（例如:alert(a)）,将优先调用toString方法，如若没有重写toString将调用valueOf方法，如果两方法都不没有重写，但按Object的toString输出。

2. 在进行强转字符串类型时将优先调用toString方法，强转为数字时优先调用valueOf。

3. 在有运算操作符的情况下，valueOf的优先级高于toString。

任何一个数与NaN进行关系比较，均为false

```NaN != NaN```

实际开发用不建议使用with(statement){}语句

switch语句中使用的是全等操作符

ECMAScript函数中arguments对象与数组类似（并非Array的实例），可以通过arguments[i]来访问参数。arguments.length来获取传入参数的个数，arguments唯一由传入参数决定，与形式参数无关。并且，可以通过arguments来进行函数的重载,它没有传统意义上的重载方式。
return语句可以缺省，如果没用return，默认返回一个undefined
如果定义了两个重名的函数，则该名字属于后定义的函数，后定义的函数覆盖了之前定义的函数。

> JS中两种数据类型：基本类型值（赋值是复制）和引用类型值（地址的复制）
> JS中函数的参数传递全部都是按值传递的