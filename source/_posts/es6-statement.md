---
title: ES6语句或语句块中的闭包问题详解
tags: [JavaScript,ES6]
categories: JavaScript
cover: http://bpic.588ku.com/back_pic/00/04/13/75/29f23b37c3fd039e262b957399f6ffcf.jpg
---
一般情况下，当一个函数实例被创建时，它唯一对应的一个闭包也就被创建。在下面的代码中，由于外部的构造器函数被执行两次，因此内部的foo函数也被创建了两个函数实例（以及闭包）并赋值给this对象的成员：

```javascript
function MyObject() {
    function foo() {}
    this.method = foo;
}
obj1 = new MyObject();
obj2 = new MyObject();

//显示false，表明产生了两个对象
console.log(obj1.method === obj2.method);
```
这在函数之外（语句级别）也具有完全相同的表现。在下面的例子中，多个匿名函数的实例被赋给了obj的成员：

```javascript
var obj = new Object();
for (var i = 0; i < 5; i++) {
    obj['method' + i] = function() {
        console.log(i);
    }
}
console.log(obj.method2 === obj.method3);
```
尽管是多个实例，但它们仍然是共享同一个外层函数闭包中的upvalue值——在上例中，外层函数闭包指的是全局闭包。因此下面例子所变现出来的，仍然只是闭包中对upvalue值访问的规则，而并非闭包或者函数实例的创建规则“出现了某种特例”:

```javascript
var obj = new Object();
var events = {
  m1:'clicked',
  m2:'changed'
}
for(e in events){
  obj[e] = function () {
    console.log(events[e]);
  }
}
//显示false，表明是不同的函数实例
console.log(obj.m1 === obj.m2);

//方法m1()和m2()都都出相同值
//其原因，在于两个方法都访问全局闭包中的同一个upvalue值。
obj.m1();
obj.m2();
```
在这个例子中，王法m1与m2究竟输出何值，取决于前面的for...in语句在最后一次迭代中对e的置值。某些引擎中总保证for...in的顺序与events中声明时的属性顺序一致（例如SpiderMonkey），但也有一些引擎并没有这项约定。因此上例在不同的引擎中表现的结果未必一致，但m1()与m2()输出值总是相同的。

按照这段代码的本意，应该是每个函数实例输出不同值。对这个问题的处理之一，是再添加一个外层函数，利用“在函数内保存数据”的特性来为内部函数保存一个可访问的upvalue:

```javascript
var obj = new Object();
var events = {
    m1: 'clicked',
    m2: 'changed'
}
for (e in events) {
    obj[e] = function(aValue) { //闭包lv1
        return function() { //闭包lv2
            console.log(events[aValue]);
        }
    }(e)
}

/*或者使用如下代码，在闭包内通过局部变量保存数据。
for (e in events) {
    obj[e] = function() { //闭包lv1
        var aValue = e;
        return function() { //闭包lv2
            console.log(events[aValue]);
        }
    }()
}*/

//下面将输出不同的值
obj.m1();
obj.m2();
```
由于闭包lv2引用了闭包lv1中的入口参数，因此两个闭包存在了关联关系。在obj的方法未被清除之前，两个闭包都不会被销毁，但lv1为lv2保存了一个可供访问的upvalue——这除了私有变量、函数之外，也包括它的传入参数。

很明显，上例的问题在于多了一层闭包，因此增加了系统消耗。不过这并非不可避免。我们要看清楚这个问题，其本质是一要一个地方来保存for...in列举中的每一个值e,而并非是“需要一个闭包来添加一个层”。那么，既然列举过程中产生了不同的函数实例，自然也可以将值e交给这些函数实例自己去保存：

```javascript
var obj = new Object();
var events = {
    m1: 'clicked',
    m2: 'changed'
}
for (e in events) {
    (obj[e] = function(aValue) {
        //arguments.callee指向匿名函数自身
        console.log(events[arguments.callee.aValue]);
    }).aValue = e;
}
//下面将输出不同的值
obj.m1();
obj.m2();
```
