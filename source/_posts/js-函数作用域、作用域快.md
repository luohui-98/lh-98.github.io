---
title: 函数中的作用域
date: 2021-7-18
tag: js
type: js
---

函数中的作用域
---
js有基于函数的作用域，每声明一个函数都会为其自身创建一个气泡，可以在附属气泡里获取外层的变量，但是不能从外层气泡中获取附属气泡种的变量

隐藏内部实现
---
>暴露在外面的写法，很不安全
```js
function foo(a) {
    b = a + fff(a * 2);
    console.log(b * 3);
}
function fff(a) {
    return a - 1;
}

var b;
foo(2) // 15
```

>隐藏的写法

```js
function foo(a) {
    function fff(a) {
        return a - 1;
    }
    var b;
    b = a + fff(a * 2);
    console.log(b * 3);
}
foo(2) // 15
```

>规避冲突，
可以避免同名标识符之间的冲突，避免被覆盖

方法：
1. 全局命名空间，在全局声明一个独特的变量，通常是一个对象，所有需要暴露给外界的功能都会成为这个对象的属性，而不是将自己的标识符暴露在顶级的词法作用域种。
2. 模块管理，从众多模块中挑选一个来使用，任何库都无需将标识符加入到全局作用域中。


函数作用域
---
在任意代码片段外部添加包装函数都可以将内部的变量和函数定义‘隐藏’起来，外部作用域无法访问包装函数内部的任何内容

通过定义一个函数来达到是变量在函数作用域之内，虽然可以解决问题，但是并不理想，因为这个函数名称本身污染了所在的作用域，其次是必须通过显式的调用这个函数才能运行其中的代码。


```js
var a = 2;
(function foo(){
    var a = 3;
    console.log( a ); // 3
})();
console.log( a ); // 2

```
这样函数会被当作函数表达式而不是一个标准的函数声明来处理。-- 如果function是声明中的第一个第一个词，那么就是函数声明，否则就是函数表达式。

函数表达式和函数声明最大的区别是它们的名称标识符将会绑定在何处。  foo只能在所代表的位置中被访问，外部不行。

匿名和具名
---
>匿名函数表达式
```js
setTimeout( function(){
    console.log('aaa');
},1000);
```
fun没有标识符，函数声明则是不可以省略函数名称

1. 匿名函数在栈追踪中不会显示出有意义的函数名称，使得调用很困难。
2. 如果没有函数名，需要引用的时候就只能使用已经过期的arguments.callee引用，比如在递归中
3. 匿名函数省略了对于代码的可读性/可理解性很重要的函数名

行内函数表达式
```js
setTimeout( function timeoutHandler(){
    console.log('aaa');
},1000);
```

立即执行函数表达式
---

```js
var a = 2;
(function foo(){
    var a = 3;
    console.log( a ); // 3
})();
console.log( a ); // 2
```
第一个（）将函数变成了表达式，第二个（）执行了这个函数

>IIFE代表立即执行函数表达式
```js
var a = 2;
(function IIFE(){ //传统的形式
    var a = 3;
    console.log( a ); // 3
})();
console.log( a ); // 2
```
==以上两种写法功能上是一样的==

用途：

可以传参数进去
```js
var a = 2;
(function IIFE(global){ //传统的形式
    var a = 3;
    console.log( a ); // 3
    console.log( global.a ); // 3
})(window);
console.log( a ); // 2
```

块作用域
---
除了函数能产生一个作用域气泡，for循环，if，with,try/catch等都会产生一个作用域气泡。

块作用域指的是变量和函数不仅可以属于所处的作用域，也可以属于某个代码块，通常指{..}的内部
>var

其实val一个变量无论在什么地方，都会提升到全局变量，把它写在作用域内部，只是为了风格更易读而伪装出的形式上的块作用域，如果使用这种，要确保在其他地方没有重复的使用同名的变量。

>let

ES6引入了关键字let，可以将变量绑定到所在的任意作用域中，为其声明的变量隐式的劫持了所在的块作用域。

let进行声明的不会在代码在块作用域中进行提升，声明的代码在运行前，声明并不‘存在’。
```js
{
    console.log(a); // ReferenceError(没有找到)
    let a = 2;
}

```
let 声明附属于一个新的作用域而不是当前函数作用域（也不属于全局作用域）

>const

同样可以用来创建块作用域变量，但其值是固定的，之后的任何试图修改值的操作都会引起错误。

```js
var foo = true;
if(foo){
    var a = 2;
    const b = 3;
    a = 4; // 正确
    b = 6; // 错误
}
console.log(a) // 4
console.log(b) // ReferenceError
```
任何声明在某个作用域内部的变量，都将附属于这个作用域

小思考
---

```js
a = 2;
val a;
console.log(a);//2
```
输出2的原因是：声明会提升，也就是先声明再赋值，在执行查询

```js
console.log(a); // undefind
var a = 2;

// 执行顺序

val a;
console.log(a); // undefind
a = 2;
```
原因：函数声明会提升，然后执行查询，此时是没有赋值的，所以是undefind


```js
foo(); // 不是ReferenceError，而是TypeEerror

var foo = funcyion bar() {
    //...
};
```
原因：函数表达式不会被提升

>函数优先

函数声明和变量声明都会被提升，但是一个值得注意的细节是函数会被首先提升，然后才是变量。

```js
foo(); //1
var foo;
function foo() {
    console.log(1);
}
foo = function() {
    console.log(2);
}
```
输出的是1而不是2是因为函数声明先被提升到上面，然后在执行。var foo 尽管出现在function foo... 之前，但是他是重复声明会被忽略。
>后面的声明会覆盖掉前面的声明

```js
foo(); //3
function foo() {
    console.log(1);
}
var foo = function() {
    console.log(2);
}
function foo() {
    console.log(3);
}
```




