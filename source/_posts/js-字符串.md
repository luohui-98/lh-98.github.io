---
title: js字符串
tag: js
date: 2021-8-28
---
字符串和数组比较相似，都是类数组，都有length属性以及indexOf()和concat()方法。
但是数组是可变的，字符串是不可变的。字符串不可变是指字符串的成员函数不会改变其原始值，而是创建并返回一个新的字符串。而数组的成员函数都是在其原始基础上进行操作。

许多数组函数处理字符串很方便。虽然字符串没有这些函数，但是可以通过‘借用’数组的方法来处理字符串：
```js
let a ='12345'
a.map // undefined
a.join // undefined

let b = Array.prototype.join.call(a,'-')
console.log(b) // '1-2-3-4-5'
let c = Array.prototype.map.call(a, item => {
  return item + '牙牙乐'
}).join('')
console.log(c)
```
