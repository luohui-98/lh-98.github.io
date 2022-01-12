---
title: Promise
date: 2021-7-20
tga: 异步
---
## promise是什么？
1、主要用于异步计算
2、可以将异步操作队列化，按照期望的顺序执行，返回符合预期的结果
3、可以在对象之间传递和操作promise，帮助我们处理队列



## 异步回调的问题

+ 之前处理异步是通过纯粹的回调函数的形式进行处理
+ 很容易进入到回调地狱中，剥夺了函数return的能力
+ 问题可以解决，但是难以读懂，维护困难
+ 稍有不慎就会踏入回调地狱 - 嵌套层次深，不好维护

## new Promise
```js
new Promise(
  function (resolve, reject) {
    // 一段耗时的异步操作
    resolve('成功') // 数据处理完成
    // reject('失败') // 数据处理出错
  }
).then(
  (res) => {console.log(res)},  // 成功
  (err) => {console.log(err)} // 失败
)

```
```js
function get(data){
  new Promise(
    function (resolve, reject) {
      // 一段耗时的异步操作
      axios.get(....).then(res => {
        if(res.code === 200){
          resolve(res.data) // 数据处理完成
        } else{
          reject('失败') // 数据处理出错
        }
      }) 
    }
  )
}

get(data).then(res => {

}).catch(err => {

})
```

## Promise.all() 批量执行
Promise.all([p1, p2, p3])用于将多个promise实例，包装成一个新的Promise实例，返回的实例就是普通的promise
它接收一个数组作为参数
数组里可以是Promise对象，也可以是别的值，只有Promise会等待状态改变
当所有的子Promise都完成，该Promise完成，返回值是全部值得数组
有任何一个失败，该Promise失败，返回值是第一个失败的子Promise结果
```js
//切菜
function cutUp(){
    console.log('开始切菜。');
    var p = new Promise(function(resolve, reject){        //做一些异步操作
        setTimeout(function(){
            console.log('切菜完毕！');
            resolve('切好的菜');
        }, 1000);
    });
    return p;
}
//烧水
function boil(){
    console.log('开始烧水。');
    var p = new Promise(function(resolve, reject){        //做一些异步操作
        setTimeout(function(){
            console.log('烧水完毕！');
            resolve('烧好的水');
        }, 1000);
    });
    return p;
}
Promise.all([cutUp(), boil()])
    .then((result) => {
        console.log('准备工作完毕');
        console.log(result);
    })
```

## Promise.race() 它有任意一个完成就算完成

```js
let p1 = new Promise(resolve => {
    setTimeout(() => {
        resolve('I\`m p1 ')
    }, 1000)
});
let p2 = new Promise(resolve => {
    setTimeout(() => {
        resolve('I\`m p2 ')
    }, 2000)
});
Promise.race([p1, p2])
    .then(value => {
        console.log(value)
    })
```

## 回调地狱和promise对比
+ 回调地狱
```js
/***
   第一步：找到北京的id
   第二步：根据北京的id -> 找到北京公司的id
   第三步：根据北京公司的id -> 找到北京公司的详情
   目的：模拟链式调用、回调地狱
 ***/
 
 // 回调地狱
 // 请求第一个API: 地址在北京的公司的id
 $.ajax({
   url: 'https://www.easy-mock.com/mock/5a52256ad408383e0e3868d7/lagou/city',
   success (resCity) {
     let findCityId = resCity.filter(item => {
       if (item.id == 'c1') {
         return item
       }
     })[0].id
     $.ajax({
       //  请求第二个API: 根据上一个返回的在北京公司的id “findCityId”，找到北京公司的第一家公司的id
       url: 'https://www.easy-mock.com/mock/5a52256ad408383e0e3868d7/lagou/position-list',
       success (resPosition) {
         let findPostionId = resPosition.filter(item => {
           if(item.cityId == findCityId) {
             return item
           }
         })[0].id
         // 请求第三个API: 根据上一个API的id(findPostionId)找到具体公司，然后返回公司详情
         $.ajax({
           url: 'https://www.easy-mock.com/mock/5a52256ad408383e0e3868d7/lagou/company',
           success (resCom) {
             let comInfo = resCom.filter(item => {
               if (findPostionId == item.id) {
                 return item
               }
             })[0]
             console.log(comInfo)
           }
         })
       }
     })
   }
 })
```

```js
// Promise 写法
// 第一步：获取城市列表
const cityList = new Promise((resolve, reject) => {
  $.ajax({
    url: 'https://www.easy-mock.com/mock/5a52256ad408383e0e3868d7/lagou/city',
    success (res) {
      resolve(res)
    }
  })
})

// 第二步：找到城市是北京的id
cityList.then(res => {
  let findCityId = res.filter(item => {
    if (item.id == 'c1') {
      return item
    }
  })[0].id
  findCompanyId().then(res => {
    // 第三步（2）：根据北京的id -> 找到北京公司的id
    let findPostionId = res.filter(item => {
        if(item.cityId == findCityId) {
          return item
        }
    })[0].id
    // 第四步（2）：传入公司的id
    companyInfo(findPostionId)
  })
})

// 第三步（1）：根据北京的id -> 找到北京公司的id
function findCompanyId () {
  let aaa = new Promise((resolve, reject) => {
    $.ajax({
      url: 'https://www.easy-mock.com/mock/5a52256ad408383e0e3868d7/lagou/position-list',
      success (res) {
        resolve(res)
      }
    })
  })
  return aaa
}

// 第四步：根据上一个API的id(findPostionId)找到具体公司，然后返回公司详情
function companyInfo (id) {
  let companyList = new Promise((resolve, reject) => {
    $.ajax({
      url: 'https://www.easy-mock.com/mock/5a52256ad408383e0e3868d7/lagou/company',
      success (res) {
        let comInfo = res.filter(item => {
            if (id == item.id) {
               return item
            }
        })[0]
        console.log(comInfo)
      }
    })
  })
}
```
