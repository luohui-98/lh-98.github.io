---
title: 闭包
date: 2021-7-19
tag: js
type: js
---

>闭包是基于词法作用域书写代码时所产生的自然结果

闭包的产生：
函数在被定义的地方之外被执行就会产生闭包！！！

```js
function foo() {
    var a = 2;
    function bar(){
        console.log(a);
    }
    return bar;
}
var bza = foo();

baz(); // 2  这就是闭包！
```
通常情况下foo执行之后整个内部作用域都会被销毁，因为引擎会销毁不再使用的空间来释放内存空间。

然而闭包会阻止这一情况的发生，会让作用域依然存在，因为bar函数还在使用foo这个作用域，需要给bar在任何时候执行提供支持。所以foo的作用域不会被销毁，bar依然持有对该作用域的引用，这个引用就是闭包！

==无论通过何种手段将内部函数传递到所在的词法作用域以外，他都会持有对原始定义作用域的引用，无论在何处执行这个函数都会使用闭包。==

```js
function wait(message){
    setTimeout( function timer() {
        console.log(message);
    },1000)
}
wait('hello')
```
wait执行1000毫秒以后，他的内部作用域并不会消失，还能保持message的引用。

==只要是使用了回调函数，实际上就是在使用闭包。==

### 闭包和循环

```js
for(var i = 1;i <= 5; i++;){
    setTimeout(function timer() {
        console.log(i);
    },1*1000)
}
// 每秒一次输出五个6
```
这个循环终止的条件是6.条件首次成立的时候i === 6
，因此输出的显示是循环结束时i的值。
因为这里所用的i，是同一个作用域下的i，所有的函数共享一个i。

```js
for(var i = 1;i <= 5; i++;){
    (function() {
        setTimeout(function timer() {
            console.log(i);
        },1*1000)
    }();
}
//这样也不行
```
因为我们的IIFE的作用域是空的，我们使用的依然是外层的i，他要包含一点实质的内容才能够我们使用。


```js
for(var i = 1;i <= 5; i++;){
    (function(j) {
        setTimeout(function timer() {
            console.log(j);
        },j*1000)
    }(i);
}
// 这样就达到我们预期的目的，每秒一个，一次输出1-5
```

### 重返块作用域
前面说let可以劫持块级作用域，，并且在这个块级作用域中声明一个变量。看下面代码

```js
for(var i = 1; i <= 5; i++) {
    let j = i;
    setTimeout(function timer() {
        console.log(j);
    },j*1000)
};
```
还可以更完善

```js
for(let i = 1; i <= 5; i++) {
    setTimeout(function timer() {
        console.log(i);
    },i*1000)
};
```

### 模块
```js
function foo(){
    var a = 'cool';
    var b = [1,2,3];
    function bar() {
        console.log(a);
    }
    function baz() {
        console.log(b);
    }
}
```
这里并没有明显的闭包，只有两个私有数据变量a和b，以及bar和baz两个内部函数，他们的词法作用域就是闭包，
也就是foo（）的内部作用域。
```js
function CoolModule() {
    var something = 'cool';
    var another = [1,2,3];
    function doSomething() {
        console.log(something));
    }
    function doAnother() {
        console.log(another.join("!"));
    }
    return {
        doSomething: doSomething,
        doAnother: doAnother,
    };
}
var foo = CoolModule;
foo.doSomething(); // cool
foo.doAnother(); // 1!2!3
```
首先，CoolModule() 只是一个函数，必须通过调用它来创建一个模块实例，如果不执行外部函数，内部作用域和闭包都无法被创建。

其次，CoolModule() 返回一个用对象字面量语法来表示的对象，这个返回的对象中含有内部函数，而不是内部数据变量的引用。我们保持内部数据变量是隐藏且私有状态。

这个对象类型的返回值最终被赋值给外部变量foo，然后我们就可以通过它来访问API中的属性方法。 

>模块模式必须具有两个条件
1. 必须有外部的封闭函数，该函数必须至少被调用一次。（每次调用都会创建一个新的模块实例）
2. 封闭函数必须返回至少一个内部函数，这样内部函数才能在私有作用域中形成闭包，并且可以访问或者修改私有的状态。

一个具有函数属性的对象本身并不是真正的模块。从方便观察角度来看，一个从函数调用所返回的，只有数据属性而没有闭包函数的对象并不是真正的模块。

改进上面的代码
```js
var foo = (function CoolModule() {
    var something = 'cool';
    var another = [1,2,3];
    function doSomething() {
        console.log(something));
    }
    function doAnother() {
        console.log(another.join("!"));
    }
    return {
        doSomething: doSomething,
        doAnother: doAnother,
    };
})();

foo.doSomething(); // cool
foo.doAnother(); // 1!2!3
```

模块是普通函数，因此也可以传参。
```js
function CoolModule(id) {
    function doSomething() {
        console.log(id);
    }
    return {
        doSomething: doSomething,
    };
}
var foo = CoolModule('foo');
foo.doSomething(); // foo
```

>模块另一个简单但又强大的用法是命名将要作为公共API返回的对象。
```js
var foo = (function Cool(id){
    function change() {
        publicAPI.identify = identify2;
    };
    function identify1() {
        console.log(id);
    };
    function identify2() {
        console.log(id.toUppeCase());
    };
    var publicAPI = {
        change: change,
        identify: identify1
    }
    return publicAPI;
})('foo module');

foo.identify(); // foo module
foo.change();
foo.identify(); // FOO MODULE
```
通过模块在实例的内部保留的公共API对象的内部引用，可以从内部对模块实例进行修改，包括添加或删除方法，以及修改他们的值。

现代的模块机制
---
创建一个模块
```js
var MyModules = (function Manager(){
    // 存储方法
    var modules = {};
    
    // 添加方法
    function define(name,deps,impl){
        for(var i = 0;i < deps.length; i++) {
            deps[i] = modules[deps[i]];
        }
        modules[name] = impl.apply(impl,deps);
    }
    
    // 根据名字获取方法
    function get(name) {
        return modules[name];
    }
    
    // 返回内部函数（方法）
    return {
        define: define,
        get: get
    }
})()
```
Function.apply(obj,args)方法能接收两个参数

obj：这个对象将代替Function类里this对象

args：这个是数组，它将作为参数传给Function(args-->arguments)

使用它来定义模块：
```js
MyModules.define("bar",[], function() {
    function hello(who) {
        return "let me introduce:" + who;
    }
    retrun {
        hello : hello
    };
});

MyModules.define("foo",['bar'], function(bar) {
    var hungry = 'hippo';
    function awesome(who) {
        console.log(bar.hello(hungry).toUpperCase());
    }
    retrun {
        awesome : awesome
    };
});

var bar = MyModules.get('bar');
var foo = MyModules.get('foo');

console.log(bar.hello('hippo'));// let me introduce: hippo

foo.awesome(); // 大写的
```

foo 和 bar 都是通过一个返回的公共的API的函数来定义的。foo 甚至接受 bar 的实例作为依赖参数，并且相应地使用它。
