---
title: js数组
tag: js
date: 2021-8-27
---
数组可以容纳任何类型的的值，可以是字符串、数字、对象、甚至是其他数组。数组申明后就可以往里面添加值，不需要预设大小。

# 数组方法
1. push()
向数组的末尾添加新内容
参数：要添加的项。传递多个用逗号隔开，任何数据类型都可以
返回值：新增后数组的长度
是否改变原数组：改变
```js
let ary1 = [12,34,26];
ary1.push(100); //返回一个新的长度 
length=4console.log(ary1)//结果为 [12,34,26,100]
```
2. pop()
删除数组的最后一项
参数：无
返回值：被删除的项
是否改变原数组：改变
```js
let ary2 = [108,112,39,10];
ary2.pop();//删除的最后一项为10
console.log(ary2);//[108, 112, 39]
```
3. shift()
删除数组的第一项
参数：无
返回值：被删除的项
是否改变原数组：改变
```js
let ary3 = [0,108,112,39];
ary3.shift();//删除的第一项为0
console.log(ary3);//[108, 112, 39]
```
4. unshift()
向数组首位添加新内容
参数：要添加的项，多项用','隔开
返回值：新数组的长度
是否改变原数组：改变
```js
let ary4 = ['c','d'];
ary4.unshift('a','b');
console.log(ary4);//["a", "b", "c", "d"]
```
5. slice()
按照条件查找出其中的部分内容
参数：
array.slice(n, m)，从索引n开始查找到m处（不包含m）
array.slice(n) 第二个参数省略，则一直查找到末尾
array.slice(0)原样输出内容，可以实现数组克隆
array.slice(-n,-m) slice支持负参数，从最后一项开始算起，-1为最后一项，-2为倒数第二项
返回值：返回一个新数组
是否改变原数组：不改变
```js
let ary5 = [1,2,3,4,5,6,7,8,9]; 
console.log(ary5.slice(2,8));//从索引2开始查找到索引为8的内容，结果为[3, 4, 5, 6, 7, 8] 
console.log(ary5.slice(0)); 
console.log(ary5.slice(-2,-1));//[8]
```
6. splice()
对数组进行增删改
增加：ary.splice(n,0,m)从索引n开始删除0项，把m或者更多的内容插入到索引n的前面
返回空数组
修改：ary.splice(n,x,m)从索引n开始删除x个，m替换删除的部分
把原有内容删除掉，然后用新内容替换掉
删除：ary.splice(n,m) 从索引n开始删除m个内容
（如果第二个参数省略，则从n删除到末尾）
返回删除的新数组，原有数组改变
```js
//增加
  let ary6_z = [33,44,55,66,77,88];
  ary6_z.splice(2,0,'a','b')
  console.log(ary6_z);// [33, 44, "a", "b", 55, 66, 77, 88]

//修改
  let ary6_x = [33,44,55,66,77,88];
  ary6_x.splice(1,2,'x','y')
  console.log(ary6_x);// [33, "x", "y", 66, 77, 88]

//删除
   let ary6_s = [33,44,55,66,77,88];
   //console.log(ary6.splice(3,2))//[66, 77]
   console.log(ary6_s.splice(3));//[66, 77, 88]
```
7. join()
用指定的分隔符将数组每一项拼接为字符串
参数：指定的分隔符（如果省略该参数，则使用逗号作为分隔符）
返回值：拼接好的字符串
是否改变原数组：不改变
```js
let ary7 = [1,2,3];
console.log(ary7.join('、'));//1、2、3
```
8. concat()
用于连接两个或多个数组
参数：参数可以是具体的值，也可以是数组对象。可以是任意多个
返回值：返回连接后的新数组
是否改变原数组：不改变
```js
let ary8 = ['你'];
let ary80 = ary8.concat('好');
console.log(ary80);//["你", "好"]
```

9.  indexOf()
检测当前值在数组中第一次出现的位置索引
参数：array.indexOf(item,start) item:查找的元素 start:字符串中开始检索的位置
返回值：第一次查到的索引，未找到返回-1
是否改变原数组：不改变
```js
let ary9 = ['a','b','c','d','e','a','f'];   
console.log(ary9.indexOf('c'));//2
console.log(ary9.indexOf('a',3))//5
```

10. lastIndexOf()
检测当前值在数组中最后一次出现的位置索引
参数：array.lastIndexOf(item,start) item:查找的元素 start:字符串中开始检索的位置
返回值：第一次查到的索引，未找到返回-1
是否改变原数组：不改变
```js
let ary10 = ['a','b','c','d','e','a','f'];   
console.log(ary10.lastIndexOf('c'));//2
console.log(ary10.lastIndexOf('f',1))//-1
```

11. includes()
判断一个数组是否包含一个指定的值
参数：指定的内容
返回值：布尔值
是否改变原数组：不改变
```js
let ary13 = ['a','b','c','d']; 
console.log(ary13.includes('c'));//true
console.log(ary13.includes(2));//false
```

12. sort()
对数组的元素进行排序（默认是从小到大来排序 并且是根据字符串来排序的）
参数：可选(函数) 规定排序规则 默认排序顺序为按字母升序
返回值：排序后新数组
是否改变原数组：改变
sort在不传递参数情况下，只能处理10以内（个位数）数字排序
```js
let ary11 = [32,44,23,54,90,12,9]; 
ary11.sort(function(a,b){
  // return a-b;  // 结果[9, 12, 23, 32, 44, 54, 90]
  // return b-a;  // 结果[90, 54, 44, 32, 23, 12, 9]   })  
  console.log(ary11);
```
13. reverse()
把数组倒过来排列
参数：无
返回值：倒序后新数组
是否改变原数组：改变
```js
let ary12 = [6,8,10,12]; 
console.log(ary12.reverse());//[12, 10, 8, 6]
```
14. forEach()
循环遍历数组每一项
参数：函数 ary.forEach(function(item,index,ary){}) item:每一项 index:索引 ary:当前数组
返回值：无
是否改变原数组：不改变   （**但是这里我们常用forEach来改变数组里面的值，需要具体去分析**）
forEach中不能使用continue和break，forEach中不能跳出，只能跳过(return跳过)
  ```js
  let ary14 = ['a','b','c','d']; 
  let item = ary14.forEach(function(item,index,ary){
    onsole.log(item,index,ary);
  })
  ```
  + 改变原数组中引用数据类型值、不改变原数组中基本数据类型
    ```js
    let obj = {'a': 1}
    let arr = [1,2,obj,true]
    arr.forEach(item => {
      if(typeof item === 'object'){
        item['bbb'] = 2
      }
      item = '改变后'
    })
    console.log(arr) // [1,2,{'a':1,'bbb':2},true]
    ```
  + 改变原数组中基本数据类型值
  ```js
  let obj = {'a': 1}
  let arr = [1,2,obj,true]
  arr.forEach((item,key) => {
    if(typeof item === 'object'){
      arr[key]['bbb'] = '改变后'
    } else{
      arr[key] = '改变后'
    }
  })
  console.log(arr) // ["改变后", "改变后", {a: 1,bbb: "改变后"}, "改变后"]
  ```
  原因是循环出来的如果是基本数据类型的话，就会是一个值（1 = 2），这样肯定是实现不了的。

