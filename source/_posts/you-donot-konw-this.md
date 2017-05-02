---
title: JS中的This别再被问倒了，面试常见问题分析
date: 2017-05-02 12:21:31
tags: [JavaScript,ES6]
categories: JavaScript
---

GitHub地址：https://github.com/SimonZhangITer/MyBlog/issues/12

> this的指向问题应该是让每一个前端er都头疼的问题，我也一样，曾经遇到甚至都是一顿乱猜。最近在研读一些书籍如《你不知道的JavaScript》和《JavaScript语言精粹与编程实践》，让我对this的问题豁然开朗。故写下此篇文章，分享一下我的心得。

# 隐式绑定
关于this，一般来说，谁调用了方法，该方法的this就指向谁，如：

```javascript
function foo(){
	console.log(this.a)
}

var a = 3;

var obj = {
	a: 2,
	foo: foo
};

obj.foo(); // 输出2,因为是obj调用的foo，所以foo的this指向了obj，而obj.a = 2
```

如果存在多次调用，`对象属性引用链只有上一层或者说最后一层在调用位置中起作用`，如：

```javascript
function foo() {
    console.log( this.a )
}

var obj2 = {
    a: 42,
    foo: foo
}

var obj1 = {
    a: 2,
    obj2: obj2
}

obj1.obj2.foo(); // 42
```
## 隐式丢失
一个最常见的this绑定问题就是被`隐式绑定`的函数会丢失绑定对象，也就是说他回应用默认绑定，从而把this绑定到全局对象或者undefined上，取决于是否是严格模式。

```javascript
function foo() {
    console.log( this.a )
}

var obj1 = {
    a: 2,
    foo: foo
}

var bar = obj1.foo; // 函数别名！

var a = "oops, global"; // a是全局对象的属性

bar(); // "oops, global"
```
`虽然bar是obj.foo的一个引用，但是实际上，它引用的是foo函数本身，因此此时的bar()其实是一个不带任何修饰的函数调用，因此应用了默认绑定`

一个更微妙、更常见并且更出乎意料的情况发生在`传入回调函数时`：

```javascript
function foo() {
    console.log( this.a )
}

function doFoo( fn ){
    // fn 其实引用的是 foo
    fn(); // <-- 调用位置！
}

var obj = {
    a: 2,
    foo: foo
}

var a = "oops, global"; // a是全局对象的属性

doFoo( obj.foo ); // "oops, global"
```
`参数传递其实就是一种隐式赋值`，因此我们传入函数时也会被隐式赋值，所以结果和上一个例子一样，如果把函数传入语言内置的函数而不是传入自己声明的函数（如setTimeout等），`结果也是一样的`

# 显式绑定
简单的说，就是指定this，如：call、apply、bind、new绑定等

## 硬绑定
```javascript
function foo( something ) {
    console.log( this.a, something)
    return this.a + something
}

var obj = {
    a: 2
}

var bar = function() {
    return foo.apply( obj, arguments)
}

var b = bar(3); // 2 3
console.log(b); // 5
```
这里简单做一下解释：
在bar函数中，foo使用apply函数绑定了obj，也就是说foo中的this将指向obj，与此同时，使用arguments（不限制传入参数的数量）作为参数传入foo函数中；所以在运行bar(3)的时候，首先输出obj.a也就是2和传入的3，然后foo返回了两者的相加值，所以b的值为5

同样，本例也可以使用`bind`:

```javascript
function foo( something ) {
    console.log( this.a, something)
    return this.a + something
}

var obj = {
    a: 2
}

var bar = foo.bind(obj)

var b = bar(3); // 2 3
console.log(b); // 5
```
## new绑定
在传统面向类的语言中，使用new初始化类的时候会调用类中的构造函数，但是JS中new的机制实际上和面向类和语言完全不同。

使用`new`来调用函数，或者说发生构造函数调用时，会自动执行下面的操作：

- 创建（或者说构造）一个全新的对象
- 这个新对象会被执行[[Prototype]]连接
- 这个新对象会绑定到函数调用的this
- 如果函数没有返回其他对象，那么new表达式中的函数会自动返回这个新对象
如：

```javascript
function foo(a){
    this.a = a
}

var bar = new foo(2);
console.log(bar.a); // 2
```

使用new来调用foo(...)时，我们会构造一个新对象并把它绑定到foo(...)调用中的this上。new是最后一种可以影响函数调用时this绑定行为的方法，我们称之为new绑定。

# this的优先级
毫无疑问，默认绑定的优先级是四条规则中最低的，所以我们可以先不考虑它。

隐式绑定和显式绑定哪个优先级更高？我们来测试一下：

```javascript
function foo(a){
    console.log(this.a)
}

var obj1 = {
    a: 2,
    foo: foo
}

var obj2 = {
    a: 3,
    foo: foo
}

obj1.foo(); // 2
obj2.foo(); // 3

obj1.foo.call(obj2); // 3
obj2.foo.call(obj1); // 2
```
可以看到，`显式绑定`优先级更高，也就是说在判断时应当先考虑是否可以存在显式绑定。

现在我们要搞清楚`new绑定`和`隐式绑定`的优先级谁高谁低 ：

```javascript
function foo(something){
    this.a = something
}

var obj1 = {
    foo: foo
}

var obj2 = {}

obj1.foo(2);
console.log(obj1.a); // 2

obj1.foo.call(obj2,3);
console.log(obj2.a); // 3

var bar = new obj1.foo(4)
console.log(obj1.a); // 2
console.log(bar.a); // 4
```
可以看到`new绑定`比`隐式绑定`优先级高。但是`new绑定`和`显式绑定`谁的优先级更高呢？

```javascript
function foo(something){
    this.a = something
}

var obj1 = {}

var bar = foo.bind(obj1);
bar(2);
console.log(obj1.a); // 2

var baz = new bar(3);
console.log(obj1.a); // 2
console.log(baz.a); // 3
```
可以看到，`new绑定`修改了`硬绑定`中的this，所以`new绑定`的优先级比`显式绑定`更高。

之所以要在new中使用硬绑定函数，主要目的是预先设置函数的一些参数，这样在使用new进行初始化时就可以只传入其余的参数。bind(...)的功能之一就是可以把除了第一个参数（第一个参数用于绑定this）之外的其他参数都传给下层的函数（这种技术称为“部分应用”，是“`柯里化`”的一种）。举例来说：

```javascript
function foo(p1,p2){
    this.val = p1 + p2;
}

// 之所以使用null是因为在本例中我们并不关心硬绑定的this是什么
// 反正使用new时this会被修改
var bar = foo.bind(null,'p1');

var baz = new bar('p2');

baz.val; // p1p2
}
```
> 柯里化:在直觉上，柯里化声称“如果你固定某些参数，你将得到接受余下参数的一个函数”。所以对于有两个变量的函数yx，如果固定了 y = 2，则得到有一个变量的函数 2x

# This在箭头函数中的应用

箭头函数不使用this的四种标准规则，而是根据外层（函数或者全局）作用域来决定this。

我们来看一下箭头函数的词法作用域：

```javascript
function foo() {
	// 返回一个箭头函数
	return (a) => {
		// this继承自foo()
		console.log(this.a)
	};
}

var obj1 = {
	a: 2
};

var obj2 = {
	a: 3
};

var bar = foo.call(obj1);
bar.call(obj2); // 2, 不是3！
```

foo()内部创建的箭头函数会捕获调用时foo()的this。由于foo()的this绑定到obj1，bar（引用箭头函数）的this也会绑定到obj1，箭头函数的绑定无法被修改。（new也不行!）

# 总结

如果要判断一个运行中的函数的this绑定，就需要找到这个函数的直接调用位置。找到之后就可以顺序应用下面这四条规则来判断this的绑定对象。

1. 由new调用？绑定到新创建的对象。
2. 由call或者apply（或者bind）调用？绑定到指定的对象。
3. 由上下文对象调用？绑定到那个上下文对象。
4. 默认：在严格模式下绑定到undefined，否则绑定到全局对象。
