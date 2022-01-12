---
title: vue双向绑定原理
tag: vue2.0
date: 2021-7-21
---
>Vue双向绑定分为两个部分：对象和数组

# **object**双向绑定原理
## 实现方式
1. 通过Object.defineProperty()定义一个observer类，将正常的object转换成一个可观测的object，并且加上了_ob_属性，如果读取了会通过触发get，如果改变了会触发set
```js
  // 源码位置：src/core/observer/index.js

/**
 * Observer类会通过递归的方式把一个对象的所有属性都转化成可观测对象
 */
export class Observer {
  constructor (value) {
    this.value = value
    // 给value新增一个__ob__属性，值为该value的Observer实例
    // 相当于为value打上标记，表示它已经被转化成响应式了，避免重复操作
    def(value,'__ob__',this)
    if (Array.isArray(value)) {
      // 当value为数组时的逻辑
      // ...
    } else {
      this.walk(value)
    }
  }

  walk (obj: Object) {
    const keys = Object.keys(obj)
    for (let i = 0; i < keys.length; i++) {
      defineReactive(obj, keys[i])
    }
  }
}
/**
 * 使一个对象转化成可观测对象
 * @param { Object } obj 对象
 * @param { String } key 对象的key
 * @param { Any } val 对象的某个key的值
 */
function defineReactive (obj,key,val) {
  // 如果只传了obj和key，那么val = obj[key]
  if (arguments.length === 2) {
    val = obj[key]
  }
  if(typeof val === 'object'){
      new Observer(val)
  }
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get(){
      console.log(`${key}属性被读取了`);
      return val;
    },
    set(newVal){
      if(val === newVal){
          return
      }
      console.log(`${key}属性被修改了`);
      val = newVal;
    }
  })
}
```

2. 收集依赖，也就是收集那里用到了这个数据的。具体做法是定义一个dep类，其实就是一个数组，那里用到了就在数据里面push进去。dep里面有几个方法，在observer的get里面depend负责收集依赖（其实就是Watcher），在set里面notify通知所有依赖更新（通知的也是Watcher，最后由Watcher通知视图更新）
```js
// 源码位置：src/core/observer/dep.js
export default class Dep {
  constructor () {
    this.subs = []
  }

  addSub (sub) {
    this.subs.push(sub)
  }
  // 删除一个依赖
  removeSub (sub) {
    remove(this.subs, sub)
  }
  // 添加一个依赖
  depend () {
    if (window.target) {
      this.addSub(window.target)
    }
  }
  // 通知所有依赖更新
  notify () {
    const subs = this.subs.slice()
    for (let i = 0, l = subs.length; i < l; i++) {
      subs[i].update()
    }
  }
}

/**
 * Remove an item from an array
 */
export function remove (arr, item) {
  if (arr.length) {
    const index = arr.indexOf(item)
    if (index > -1) {
      return arr.splice(index, 1)
    }
  }
}
```
3. Watcher类，Watcher先把自己设置到全局唯一的指定位置（window.target），然后读取数据。因为读取了数据，所以会触发这个数据的getter。接着，在getter中就会从全局唯一的那个位置读取当前正在读取数据的Watcher，并把这个watcher收集到Dep中去。收集好之后，当数据发生变化时，会向Dep中的每个Watcher发送通知。通过这样的方式，Watcher可以主动去订阅任意一个数据的变化。
```js
export default class Watcher {
  constructor (vm,expOrFn,cb) {
    this.vm = vm;
    this.cb = cb;
    this.getter = parsePath(expOrFn)
    this.value = this.get()
  }
  get () {
    window.target = this;
    const vm = this.vm
    let value = this.getter.call(vm, vm)
    window.target = undefined;
    return value
  }
  update () {
    const oldValue = this.value
    this.value = this.get()
    this.cb.call(this.vm, this.value, oldValue)
  }
}

/**
 * Parse simple path.
 * 把一个形如'data.a.b.c'的字符串路径所表示的值，从真实的data对象中取出来
 * 例如：
 * data = {a:{b:{c:2}}}
 * parsePath('a.b.c')(data)  // 2
 */
const bailRE = /[^\w.$]/
export function parsePath (path) {
  if (bailRE.test(path)) {
    return
  }
  const segments = path.split('.')
  return function (obj) {
    for (let i = 0; i < segments.length; i++) {
      if (!obj) return
      obj = obj[segments[i]]
    }
    return obj
  }
}
```

## 总结

Data通过observer转换成了getter/setter的形式来追踪变化。
当外界通过Watcher读取数据时，会触发getter从而将Watcher添加到依赖中。
当数据发生了变化时，会触发setter，从而向Dep中的依赖（即Watcher）发送通知。
Watcher接收到通知后，会向外界发送通知，变化通知到外界后可能会触发视图更新，也有可能触发用户的某个回调函数等。

## 不足

虽然我们通过Object.defineProperty方法实现了对object数据的可观测，但是这个方法仅仅只能观测到object数据的取值及设置值，当我们向object数据里添加一对新的key/value或删除一对已有的key/value时，它是无法观测到的，导致当我们对object数据添加或删除值时，无法通知依赖，无法驱动视图进行响应式更新。

当然，Vue也注意到了这一点，为了解决这一问题，Vue增加了两个全局API:Vue.set和Vue.delete，这两个API的实现原理将会在后面学习全局API的时候说到。


# **Array**双向绑定原理
为什么Object数据和Array型数据会有两种不同的变化侦测方式？

这是因为对于Object数据我们使用的是JS提供的对象原型上的方法Object.defineProperty，而这个方法是对象原型上的，所以Array无法使用这个方法，所以我们需要对Array型数据设计一套另外的变化侦测机制。

万变不离其宗，虽然对Array型数据设计了新的变化侦测机制，但是其根本思路还是不变的。那就是：还是在获取数据时收集依赖，数据变化时通知依赖更新。
## 实现方式
1. 收集依赖
```js
data(){
  return {
    arr:[1,2,3]
  }
}
```
arr这个数据始终都存在于一个object数据对象中，而且我们也说了，谁用到了数据谁就是依赖，那么要用到arr这个数据，是不是得先从object数据对象中获取一下arr数据，而从object数据对象中获取arr数据自然就会触发arr的getter，所以我们就可以在getter中收集依赖。
总结一句话就是：**Array型数据还是在getter中收集依赖。**

2. 使Array型数据可观测
```js
let arr = [1,2,3]
arr.push(4)
Array.prototype.newPush = function(val){
  console.log('arr被修改了')
  this.push(val)
}
arr.newPush(4)
```
在上面这个例子中，我们针对数组的原生push方法定义个一个新的newPush方法，这个newPush方法内部调用了原生push方法，这样就保证了新的newPush方法跟原生push方法具有相同的功能，而且我们还可以在新的newPush方法内部干一些别的事情，比如通知变化
 ### 数组方法拦截器
 在Vue中创建了一个数组方法拦截器，它拦截在数组实例与Array.prototype之间，在拦截器内重写了操作数组的一些方法，当数组实例使用操作数组方法时，其实使用的是拦截器中重写的方法，而不再使用Array.prototype上的原生方法
![Image text](/imgs/vue1.png)

```js
const arrayProto = Array.prototype
// 创建一个对象作为拦截器
export const arrayMethods = Object.create(arrayProto)

// 改变数组自身内容的7个方法
const methodsToPatch = [
  'push',
  'pop',
  'shift',
  'unshift',
  'splice',
  'sort',
  'reverse'
]

/**
 * Intercept mutating methods and emit events
 */
methodsToPatch.forEach(function (method) {
  const original = arrayProto[method]      // 缓存原生方法
  Object.defineProperty(arrayMethods, method, {
    enumerable: false,
    configurable: true,
    writable: true,
    value:function mutator(...args){
      const result = original.apply(this, args)
      return result
    }
  })
})
```
在上面的代码中，首先创建了继承自Array原型的空对象arrayMethods，接着在arrayMethods上使用object.defineProperty方法将那些可以改变数组自身的7个方法遍历逐个进行封装。最后，当我们使用push方法的时候，其实用的是arrayMethods.push，而arrayMethods.push就是封装的新函数mutator，也就后说，实标上执行的是函数mutator，而mutator函数内部执行了original函数，这个original函数就是Array.prototype上对应的原生方法。 那么，接下来我们就可以在mutator函数中做一些其他的事，比如说发送变化通知。

### 使用拦截器
```js
// 源码位置：/src/core/observer/index.js
export class Observer {
  constructor (value) {
    this.value = value
    if (Array.isArray(value)) {
      const augment = hasProto
        ? protoAugment
        : copyAugment
      augment(value, arrayMethods, arrayKeys)
    } else {
      this.walk(value)
    }
  }
}
// 能力检测：判断__proto__是否可用，因为有的浏览器不支持该属性
export const hasProto = '__proto__' in {}

const arrayKeys = Object.getOwnPropertyNames(arrayMethods)

/**
 * Augment an target Object or Array by intercepting
 * the prototype chain using __proto__
 */
function protoAugment (target, src: Object, keys: any) {
  target.__proto__ = src
}

/**
 * Augment an target Object or Array by defining
 * hidden properties.
 */
/* istanbul ignore next */
function copyAugment (target: Object, src: Object, keys: Array<string>) {
  for (let i = 0, l = keys.length; i < l; i++) {
    const key = keys[i]
    def(target, key, src[key])
  }
}

```

### 依赖收集
```js
// 源码位置：/src/core/observer/index.js
export class Observer {
  constructor (value) {
    this.value = value
    this.dep = new Dep()    // 实例化一个依赖管理器，用来收集数组依赖
    if (Array.isArray(value)) {
      const augment = hasProto
        ? protoAugment
        : copyAugment
      augment(value, arrayMethods, arrayKeys)
    } else {
      this.walk(value)
    }
  }
}
```
```js
function defineReactive (obj,key,val) {
  let childOb = observe(val)
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get(){
      if (childOb) {
        childOb.dep.depend()
      }
      return val;
    },
    set(newVal){
      if(val === newVal){
        return
      }
      val = newVal;
      dep.notify()   // 在setter中通知依赖更新
    }
  })
}

/**
 * Attempt to create an observer instance for a value,
 * returns the new observer if successfully observed,
 * or the existing observer if the value already has one.
 * 尝试为value创建一个0bserver实例，如果创建成功，直接返回新创建的Observer实例。
 * 如果 Value 已经存在一个Observer实例，则直接返回它
 */
export function observe (value, asRootData){
  if (!isObject(value) || value instanceof VNode) {
    return
  }
  let ob
  if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
    ob = value.__ob__
  } else {
    ob = new Observer(value)
  }
  return ob
}
```

### 通知依赖
我们只要能访问到被转化成响应式的数据value即可，因为vaule上的__ob__就是其对应的Observer类实例，有了Observer类实例我们就能访问到它上面的依赖管理器，然后只需调用依赖管理器的dep.notify()方法，让它去通知依赖更新即可
```js
/**
 * Intercept mutating methods and emit events
 */
methodsToPatch.forEach(function (method) {
  const original = arrayProto[method]
  def(arrayMethods, method, function mutator (...args) {
    const result = original.apply(this, args)
    const ob = this.__ob__
    // notify change
    ob.dep.notify()
    return result
  })
})
```

## 不足
前文中我们说过，对于数组变化侦测是通过拦截器实现的，也就是说只要是通过数组原型上的方法对数组进行操作就都可以侦测到，但是别忘了，我们在日常开发中，还可以通过数组的下标来操作数据，如下
```js
let arr = [1,2,3]
arr[0] = 5;       // 通过数组下标修改数组中的数据
arr.length = 0    // 通过修改数组长度清空数组
```
而使用上述例子中的操作方式来修改数组是无法侦测到的。 同样，Vue也注意到了这个问题， 为了解决这一问题，Vue增加了两个全局API:Vue.set和Vue.delete，这两个API的实现原理将会在后面学习全局API的时候说到

## 总结
首先我们分析了对于Array型数据也在getter中进行依赖收集；其次我们发现，当数组数据被访问时我们轻而易举可以知道，但是被修改时我们却很难知道，为了解决这一问题，我们创建了数组方法拦截器，从而成功的将数组数据变的可观测。接着我们对数组的依赖收集及数据变化如何通知依赖进行了深入分析；最后我们发现Vue不但对数组自身进行了变化侦测，还对数组中的每一个元素以及新增的元素都进行了变化侦测
