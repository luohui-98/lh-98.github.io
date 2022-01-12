---
title: vue实例方法
tag: vue2.0
data: 2021-8-2
---
+ 数据相关的方法
  + vm.$set
  + vm.$delete
  + vm.$watch
+ 事件相关的方法
  + vm.$on
  + vm.$emit
  + vm.$off
  + vm.$once
+ 生命周期相关的方法
  + vm.$mount
  + vm.$forceUpdate
  + vm.$nextTick
  + vm.$destory

# 数据相关的方法

与数据相关的实例方法有3个，分别是vm.$set、vm.$delete和vm.$watch。它们是在stateMixin函数中挂载到Vue原型上的，代码如下：
```js
import {set,del} from '../observer/index'
// 当执行stateMixin函数后，会向Vue原型上挂载上述3个实例方法
export function stateMixin (Vue) {
    Vue.prototype.$set = set
    Vue.prototype.$delete = del
    Vue.prototype.$watch = function (expOrFn,cb,options) {}
}
```
##  vm.$watch
### 用法
```js
vm.$watch( expOrFn, callback, [options] )
```
+ 参数
  + {string | Function} expOrFn
  + {Function | Object} callback
  + {Object} [options]
    + {boolean} deep
    + {boolean} immediate
+ 返回值
  {Function} unwatch
+ 用法
  观察 Vue 实例变化的一个表达式或计算属性函数。回调函数得到的参数为新值和旧值。表达式只接受监督的键路径。对于更复杂的表达式，用一个函数取代。
  注意：**在变异 (不是替换) 对象或数组时，旧值将与新值相同，因为它们的引用指向同一个对象/数组。Vue 不会保留变异之前值的副本。**
+ 示例
```js
// 键路径
vm.$watch('a.b.c', function (newVal, oldVal) {
  // 做点什么
})

// 函数
vm.$watch(
  function () {
    // 表达式 `this.a + this.b` 每次得出一个不同的结果时
    // 处理函数都会被调用。
    // 这就像监听一个未被定义的计算属性
    return this.a + this.b
  },
  function (newVal, oldVal) {
    // 做点什么
  }
)
```
vm.$watch 返回一个取消观察函数，用来停止触发回调：
```js
var unwatch = vm.$watch('a', cb)
// 之后取消观察
unwatch()
```
+ 选项：deep
  为了发现对象内部值的变化，可以在选项参数中指定 deep: true 。注意监听数组的变动不需要这么做。
```js
vm.$watch('someObject', callback, {
  deep: true
})
vm.someObject.nestedValue = 123
// callback is fired
```
+ 选项：immediate
  在选项参数中指定 immediate: true 将立即以表达式的当前值触发回调：
```js
vm.$watch('a', callback, {
  immediate: true
})
// 立即以 `a` 的当前值触发回调
```
注意在带有 immediate 选项时，你不能在第一次回调时取消侦听给定的 property。
```js
// 这会导致报错
var unwatch = vm.$watch(
  'value',
  function () {
    doSomething()
    unwatch()
  },
  { immediate: true }
)
```
如果你仍然希望在回调内部调用一个取消侦听的函数，你应该先检查其函数的可用性：
```js
var unwatch = vm.$watch(
  'value',
  function () {
    doSomething()
    if (unwatch) {
      unwatch()
    }
  },
  { immediate: true }
)
```
###  内部原理
$watch的定义位于源码的src/core/instance/state.js中，如下：
```js
Vue.prototype.$watch = function (expOrFn,cb,options) {
    const vm: Component = this
    if (isPlainObject(cb)) {
      return createWatcher(vm, expOrFn, cb, options)
    }
    options = options || {}
    options.user = true
    const watcher = new Watcher(vm, expOrFn, cb, options)
    if (options.immediate) {
      cb.call(vm, watcher.value)
    }
    return function unwatchFn () {
      watcher.teardown()
    }
  }
```
在函数内部，首先判断传入的回调函数是否为一个对象，就像下面这种形式：
```js
vm.$watch(
    'a.b.c',
    {
        handler: function (val, oldVal) { /* ... */ },
        deep: true
    }
)
```
如果传入的回调函数是个对象，那就表明用户是把第二个参数回调函数cb和第三个参数选项options合起来传入的，此时调用createWatcher函数，该函数定义如下：
```js
function createWatcher (vm,expOrFn,handler,options) {
    if (isPlainObject(handler)) {
        options = handler
        handler = handler.handler
    }
    if (typeof handler === 'string') {
        handler = vm[handler]
    }
    return vm.$watch(expOrFn, handler, options)
}
```
可以看到，该函数内部其实就是从用户合起来传入的对象中把回调函数cb和参数options剥离出来，然后再以常规的方式调用$watch方法并将剥离出来的参数穿进去
接着获取到用户传入的options，如果用户没有传入则将其赋值为一个默认空对象，如下
```js
options = options || {}
```
$watch方法内部会创建一个watcher实例，由于该实例是用户手动调用$watch方法创建而来的，所以给options添加user属性并赋值为true，用于区分用户创建的watcher实例和Vue内部创建的watcher实例，如下：
```js
options.user = true
```
接着，传入参数创建一个watcher实例，如下：
```js
const watcher = new Watcher(vm, expOrFn, cb, options)
```
接着判断如果用户在选项参数options中指定的immediate为true，则立即用被观察数据当前的值触发回调，如下：
```js
if (options.immediate) {
    cb.call(vm, watcher.value)
}
```
最后返回一个取消观察函数unwatchFn，用来停止触发回调。如下：
```js
return function unwatchFn () {
    watcher.teardown()
}
```
这个取消观察函数unwatchFn内部其实是调用了watcher实例的teardown方法，那么我们来看一下这个teardown方法是如何实现的。其代码如下：
```js
export default class Watcher {
    constructor (/* ... */) {
        // ...
        this.deps = []
    }
    teardown () {
        let i = this.deps.length
        while (i--) {
            this.deps[i].removeSub(this)
        }
    }
}
```
在之前介绍变化侦测篇的文章中我们说过，谁读取了数据，就表示谁依赖了这个数据，那么谁就会存在于这个数据的依赖列表中，当这个数据变化时，就会通知谁。也就是说，如果谁不想依赖这个数据了，那么只需从这个数据的依赖列表中把谁删掉即可。

在上面代码中，创建watcher实例的时候会读取被观察的数据，读取了数据就表示依赖了数据，所以watcher实例就会存在于数据的依赖列表中，同时watcher实例也记录了自己依赖了哪些数据，另外我们还说过，每个数据都有一个自己的依赖管理器dep，watcher实例记录自己依赖了哪些数据其实就是把这些数据的依赖管理器dep存放在watcher实例的this.deps = []属性中，当取消观察时即watcher实例不想依赖这些数据了，那么就遍历自己记录的这些数据的依赖管理器，告诉这些数据可以从你们的依赖列表中把我删除了。

```js
vm.$watch(
    function () {
        return this.a + this.b
    },
    function (newVal, oldVal) {
        // 做点什么
    }
)
```
例如上面watcher实例，它观察了数据a和数据b，那么它就依赖了数据a和数据b，那么这个watcher实例就存在于数据a和数据b的依赖管理器depA和depB中，同时watcher实例的deps属性中也记录了这两个依赖管理器，即this.deps=[depA,depB]，

当取消观察时，就遍历this.deps，让每个依赖管理器调用其removeSub方法将这个watcher实例从自己的依赖列表中删除。

下面还有最后一个问题，当选项参数options中的deep属性为true时，如何实现深度观察呢？

首先我们来看看什么是深度观察，假如有如下被观察的数据：
```js
obj = {
    a:2
}
```
所谓深度观察，就是当obj对象发生变化时我们会得到通知，通知当obj.a属性发生变化时我们也要能得到通知，简单的说就是观察对象内部值的变化。

在创建watcher实例的时候把obj对象内部所有的值都递归的读一遍，那么这个watcher实例就会被加入到对象内所有值的依赖列表中，之后当对象内任意某个值发生变化时就能够得到通知了。

在创建watcher实例的时候，会执行Watcher类中get方法来读取一下被观察的数据，如下：
```js
export default class Watcher {
    constructor (/* ... */) {
        // ...
        this.value = this.get()
    }
    get () {
		// ...
        // "touch" every property so they are all tracked as
      	// dependencies for deep watching
        if (this.deep) {
            traverse(value)
        }
        return value
    }
}
```
可以看到，在get方法中，如果传入的deep为true，则会调用traverse函数



## vm.$set 
vm.$set 是全局 Vue.set 的别名，其用法相同。

### 用法
```js
vm.$set( target, propertyName/index, value )
```
+ 参数
  + {Object | Array} target
  + {string | number} propertyName/index
  + {any} value
+ 返回值: 设置的值
+ 用法
  向响应式对象中添加一个属性，并确保这个新属性同样是响应式的，且触发视图更新。它必须用于向响应式对象上添加新属性，因为 Vue 无法探测普通的新增属性 (比如 this.myObject.newProperty = 'hi')
+ 注意
  对象不能是 Vue 实例，或者 Vue 实例的根数据对象

### 内部原理
对于object型数据，当我们向object数据里添加一对新的key/value或删除一对已有的key/value时，Vue是无法观测到的；而对于Array型数据，当我们通过数组下标修改数组中的数据时，Vue也是是无法观测到的；

正是因为存在这个问题，所以Vue设计了set和delete这两个方法来解决这一问题，下面我们就先来看看set方法的内部实现原理。

set方法的定义位于源码的src/core/observer/index.js中，如下：
```js
export function set (target, key, val){
    if (process.env.NODE_ENV !== 'production' &&
        (isUndef(target) || isPrimitive(target))
       ) {
        warn(`Cannot set reactive property on undefined, null, or primitive value: ${(target: any)}`)
    }
    if (Array.isArray(target) && isValidArrayIndex(key)) {
        target.length = Math.max(target.length, key)
        target.splice(key, 1, val)
        return val
    }
    if (key in target && !(key in Object.prototype)) {
        target[key] = val
        return val
    }
    const ob = (target: any).__ob__
    if (target._isVue || (ob && ob.vmCount)) {
        process.env.NODE_ENV !== 'production' && warn(
            'Avoid adding reactive properties to a Vue instance or its root $data ' +
            'at runtime - declare it upfront in the data option.'
        )
        return val
    }
    if (!ob) {
        target[key] = val
        return val
    }
    defineReactive(ob.value, key, val)
    ob.dep.notify()
    return val
}
```
可以看到，方法内部的逻辑并不复杂，就是根据不同的情况作出不同的处理。
+ 首先判断在非生产环境下如果传入的target是否为undefined、null或是原始类型，如果是，则抛出警告.
+ 接着判断如果传入的target是数组并且传入的key是有效索引的话，那么就取当前数组长度与key这两者的最大值作为数组的新长度，然后使用数组的splice方法将传入的索引key对应的val值添加进数组。这里注意一点，为什么要用splice方法呢？还记得我们在介绍Array类型数据的变化侦测方式时说过，数组的splice方法已经被我们创建的拦截器重写了，也就是说，当使用splice方法向数组内添加元素时，该元素会自动被变成响应式的.
+ 如果传入的target不是数组，那就当做对象来处理。首先判断传入的key是否已经存在于target中，如果存在，表明这次操作不是新增属性，而是对已有的属性进行简单的修改值，那么就只修改属性值即可
+ 接下来获取到traget的__ob__属性，我们说过，该属性是否为true标志着target是否为响应式对象，接着判断如果tragte是 Vue 实例，或者是 Vue 实例的根数据对象，则抛出警告并退出程序
+ 接着判断如果ob属性为false，那么表明target不是一个响应式对象，那么我们只需简单给它添加上新的属性，不用将新属性转化成响应式
+ 最后，如果target是对象，并且是响应式，那么就调用defineReactive方法将新属性值添加到target上，defineReactive方会将新属性添加完之后并将其转化成响应式，最后通知依赖更新

## vm.$delete
```js
vm.$delete( target, propertyName/index )
```
+ 参数
  + {Object | Array} target
  + {string | number} propertyName/index
+ 用法
  删除对象的属性。如果对象是响应式的，确保删除能触发更新视图。这个方法主要用于避开 Vue 不能检测到属性被删除的限制，但是你应该很少会使用它。
+ 注意
  目标对象不能是一个 Vue 实例或 Vue 实例的根数据对象

### 内部原理
```js
export function del (target, key) {
  if (process.env.NODE_ENV !== 'production' &&
    (isUndef(target) || isPrimitive(target))
  ) {
    warn(`Cannot delete reactive property on undefined, null, or primitive value: ${(target: any)}`)
  }
  if (Array.isArray(target) && isValidArrayIndex(key)) {
    target.splice(key, 1)
    return
  }
  const ob = (target: any).__ob__
  if (target._isVue || (ob && ob.vmCount)) {
    process.env.NODE_ENV !== 'production' && warn(
      'Avoid deleting properties on a Vue instance or its root $data ' +
      '- just set it to null.'
    )
    return
  }
  if (!hasOwn(target, key)) {
    return
  }
  delete target[key]
  if (!ob) {
    return
  }
  ob.dep.notify()
}
```
> 该方法的内部原理与set方法有几分相似，都是根据不同情况作出不同处理。

+ 首先判断在非生产环境下如果传入的target不存在，或者target是原始值，则抛出警告
+ 接着判断如果传入的target是数组并且传入的key是有效索引的话，就使用数组的splice方法将索引key对应的值删掉，为什么要用splice方法上文中也解释了，就是因为数组的splice方法已经被我们创建的拦截器重写了，所以使用该方法会自动通知相关依赖。
+ 如果传入的target不是数组，那就当做对象来处理。
+ 接下来获取到traget的__ob__属性，我们说过，该属性是否为true标志着target是否为响应式对象，接着判断如果tragte是 Vue 实例，或者是 Vue 实例的根数据对象，则抛出警告并退出程序.
+ 接着判断传入的key是否存在于target中，如果key本来就不存在于target中，那就不用删除，直接退出程序即可
+ 最后，如果target是对象，并且传入的key也存在于target中，那么就从target中将该属性删除，同时判断当前的target是否为响应式对象，如果是响应式对象，则通知依赖更新；如果不是，删除完后直接返回不通知更新

# 事件相关的方法
与事件相关的实例方法有4个，分别是vm.$on、vm.$emit、vm.$off和vm.$once。它们是在eventsMixin函数中挂载到Vue原型上的
```js
export function eventsMixin (Vue) {
    Vue.prototype.$on = function (event, fn) {}
    Vue.prototype.$once = function (event, fn) {}
    Vue.prototype.$off = function (event, fn) {}
    Vue.prototype.$emit = function (event) {}
}
```
## vm.$on
```js
vm.$on( event, callback )
```
+ 参数
  + {string | Array<string>} event (数组只在 2.2.0+ 中支持)
  + {Function} callback
+ 作用
  监听当前实例上的自定义事件。事件可以由vm.$emit触发。回调函数会接收所有传入事件触发函数的额外参数。
+ 示例
  ```js
  vm.$on('test', function (msg) {
    console.log(msg)
  })
  vm.$emit('test', 'hi')
  // => "hi"
  ```
  ### 内部原理
  >$on和$emit这两个方法的内部原理是设计模式中最典型的发布订阅模式，首先定义一个事件中心，通过$on订阅事件，将事件存储在事件中心里面，然后通过$emit触发事件中心里面存储的订阅事件。
该方法的定义位于源码的src/core/instance/event.js中，如下：
```js
Vue.prototype.$on = function (event, fn) {
    const vm: Component = this
    if (Array.isArray(event)) {
        for (let i = 0, l = event.length; i < l; i++) {
            this.$on(event[i], fn)
        }
    } else {
        (vm._events[event] || (vm._events[event] = [])).push(fn)
    }
    return vm
}
```
$on方法接收两个参数，第一个参数是订阅的事件名，可以是数组，表示订阅多个事件。第二个参数是回调函数，当触发所订阅的事件时会执行该回调函数。

+ 首先，判断传入的事件名是否是一个数组，如果是数组，就表示需要一次性订阅多个事件，就遍历该数组，将数组中的每一个事件都递归调用$on方法将其作为单个事件订阅。
+ 如果不是数组，那就当做单个事件名来处理，以该事件名作为key，先尝试在当前实例的_events属性中获取其对应的事件列表，如果获取不到就给其赋空数组为默认值，并将第二个参数回调函数添加进去。
+ 那么问题来了，当前实例的_events属性是干嘛的呢？
  在介绍生命周期初始化阶段的初始化事件initEvents函数中，在该函数中，首先在当前实例上绑定了_events属性并给其赋值为空对象，
  ```js
  export function initEvents (vm: Component) {
    vm._events = Object.create(null)
    // ...
  }
  ```
  这个_events属性就是用来作为当前实例的事件中心，所有绑定在这个实例上的事件都会存储在事件中心_events属性中。

## vm.$emit
```js
vm.$emit( eventName, […args] )
```
+ 参数
  + {string} eventName
  + [...args]
+ 作用
   触发当前实例上的事件。附加参数都会传给监听器回调。
### 内部原理
该方法接收的第一个参数是要触发的事件名，之后的附加参数都会传给被触发事件的回调函数。
```js
Vue.prototype.$emit = function (event: string): Component {
    const vm: Component = this
    let cbs = vm._events[event]
    if (cbs) {
      cbs = cbs.length > 1 ? toArray(cbs) : cbs
      const args = toArray(arguments, 1)
      for (let i = 0, l = cbs.length; i < l; i++) {
        try {
          cbs[i].apply(vm, args)
        } catch (e) {
          handleError(e, vm, `event handler for "${event}"`)
        }
      }
    }
    return vm
  }
}
```
1. 根据传入的事件名从当前实例的_events属性（即事件中心）中获取到该事件名所对应的回调函数cbs
2. 然后再获取传入的附加参数args
3. 由于cbs是一个数组，所以遍历该数组，拿到每一个回调函数，执行回调函数并将附加参数args传给该回调。

## vm.$off
```js
vm.$off( [event, callback] )
```
+ 参数
  + {string | Array<string>} event (只在 2.2.2+ 支持数组)
  + {Function} [callback]
+ 作用
  移除自定义事件监听器。
  + 如果没有提供参数，则移除所有的事件监听器；
  + 如果只提供了事件，则移除该事件所有的监听器；
  + 如果同时提供了事件与回调，则只移除这个回调的监听器
###  内部原理
通过用法回顾我们知道，该方法用来移除事件中心里面某个事件的回调函数，根据所传入参数的不同，作出不同的处理。
```js
Vue.prototype.$off = function (event, fn) {
    const vm: Component = this
    // all
    if (!arguments.length) {
        vm._events = Object.create(null)
        return vm
    }
    // array of events
    if (Array.isArray(event)) {
        for (let i = 0, l = event.length; i < l; i++) {
            this.$off(event[i], fn)
        }
        return vm
    }
    // specific event
    const cbs = vm._events[event]
    if (!cbs) {
        return vm
    }
    if (!fn) {
        vm._events[event] = null
        return vm
    }
    if (fn) {
        // specific handler
        let cb
        let i = cbs.length
        while (i--) {
            cb = cbs[i]
            if (cb === fn || cb.fn === fn) {
                cbs.splice(i, 1)
                break
            }
        }
    }
    return vm
}
```
1. 首先，判断如果没有传入任何参数（即arguments.length为0），这就是第一种情况：如果没有提供参数，则移除所有的事件监听器。我们知道，当前实例上的所有事件都存储在事件中心_events属性中，要想移除所有的事件，那么只需把_events属性重新置为空对象即可。
2. 判断如果传入的需要移除的事件名是一个数组，就表示需要一次性移除多个事件，那么我们只需同订阅多个事件一样，遍历该数组，然后将数组中的每一个事件都递归调用$off方法进行移除即可。
3. 获取到需要移除的事件名在事件中心中对应的回调函数cbs
4. 判断如果cbs不存在，那表明在事件中心从来没有订阅过该事件，那就谈不上移除该事件，直接返回，退出程序即可。
5. 如果cbs存在，但是没有传入回调函数fn，这就是第二种情况：如果只提供了事件，则移除该事件所有的监听器。这个也不难，我们知道，在事件中心里面，一个事件名对应的回调函数是一个数组，要想移除所有的回调函数我们只需把它对应的数组设置为null即可。
6. 如果既传入了事件名，又传入了回调函数，cbs也存在，那这就是第三种情况：如果同时提供了事件与回调，则只移除这个回调的监听器。那么我们只需遍历所有回调函数数组cbs，如果cbs中某一项与fn相同，或者某一项的fn属性与fn相同，那么就将其从数组中删除即可。

## vm.$once
```js
vm.$once( event, callback )
```
+ 参数
  + {string} event
  + {Function} callback
+ 作用
  监听一个自定义事件，但是只触发一次。一旦触发之后，监听器就会被移除。

### 内部原理
该方法的作用是先订阅事件，但是该事件只能触发一次，也就是说当该事件被触发后会立即移除。我们可以定义一个子函数，用这个子函数来替换原本订阅事件所对应的回调，也就是说当触发订阅事件时，其实执行的是这个子函数，然后再子函数内部先把该订阅移除，再执行原本的回调，以此来达到只触发一次的目的。
```js
Vue.prototype.$once = function (event, fn) {
    const vm: Component = this
    function on () {
        vm.$off(event, on)
        fn.apply(vm, arguments)
    }
    on.fn = fn
    vm.$on(event, on)
    return vm
}
```
+ 可以看到，在上述代码中，被监听的事件是event，其原本对应的回调是fn，然后定义了一个子函数on。
+ 在该函数内部，先通过$on方法订阅事件，同时所使用的回调函数并不是原本的fn而是子函数on
+ 也就是说，当事件event被触发时，会执行子函数on
+ 然后在子函数内部先通过$off方法移除订阅的事件，这样确保该事件不会被再次触发，接着执行原本的回调fn
+ 另外，还有一行代码on.fn = fn是干什么的呢？上文我们说了，我们用子函数on替换了原本的订阅事件所对应的回调fn，那么在事件中心_events属性中存储的该事件名就会变成如下这个样子：
  ```js
  vm._events = {
    'xxx':[on]
  }
  ```
  但是用户自己却不知道传入的fn被替换了，当用户在触发该事件之前想调用$off方法移除该事件时
  ```js
  vm.$off('xxx',fn)
  ```
  此时就会出现问题，因为在_events属性中的事件名xxx对应的回调函数列表中没有fn，那么就会移除失败。这就让用户费解了，用户明明给xxx事件传入的回调函数是fn，现在反而找不到fn导致事件移除不了了。
  所以，为了解决这一问题，我们需要给on上绑定一个fn属性，属性值为用户传入的回调fn，这样在使用$off移除事件的时候，$off内部会判断如果回调函数列表中某一项的fn属性与fn相同时，就可以成功移除事件了。


# 生命周期相关的方法
与生命周期相关的实例方法有4个，分别是vm.$mount、vm.$forceUpdate、vm.$nextTick和vm.$destory。其中，$forceUpdate和$destroy方法是在lifecycleMixin函数中挂载到Vue原型上的，$nextTick方法是在renderMixin函数中挂载到Vue原型上的，而$mount方法是在跨平台的代码中挂载到Vue原型上的
```js
export function lifecycleMixin (Vue) {
    Vue.prototype.$forceUpdate = function () {}
    Vue.prototype.$destroy = function (fn) {}
}

export function renderMixin (Vue) {
    Vue.prototype.$nextTick = function (fn) {}
}
```
## vm.$mount
```js
vm.$mount( [elementOrSelector] )
```
+ 参数
  + {Element | string} [elementOrSelector]
  + {boolean} [hydrating]
+ 返回值: vm - 实例自身
+ 作用
  如果 Vue 实例在实例化时没有收到 el 选项，则它处于“未挂载”状态，没有关联的 DOM 元素。可以使用 vm.$mount() 手动地挂载一个未挂载的实例。
  如果没有提供 elementOrSelector 参数，模板将被渲染为文档之外的的元素，并且你必须使用原生 DOM API把它插入文档中。
  这个方法返回实例自身，因而可以链式调用其它实例方法
### 原理
关于该方法的内部原理在介绍生命周期篇的**模板编译阶段**中已经详细分析过

## vm.$forceUpdate
```js
vm.$forceUpdate()
```
+ 作用
  迫使 Vue 实例重新渲染。注意它仅仅影响实例本身和插入插槽内容的子组件，而不是所有子组件。
### 内部原理
当调用了该方法，当前实例会立即重新渲染。

**什么情况下实例会重新渲染？**那就是当实例依赖的数据发生变化时，变化的数据会通知其收集的依赖列表中的依赖进行更新，在之前的文章中我们说过，收集依赖就是收集watcher，依赖更新就是watcher调用update方法更新，所以实例依赖的数据发生变化时，就会通知实例watcher去执行update方法进行更新。

那么我们就知道了，实例的重新渲染其实就是实例watcher执行了update方法。
```js
Vue.prototype.$forceUpdate = function () {
    const vm: Component = this
    if (vm._watcher) {
        vm._watcher.update()
    }
}
```
当前实例的_watcher属性就是该实例的watcher，所以要想让实例重新渲染，我们只需手动的去执行一下实例watcher的update方法即可

## vm.$nextTick
vm.$nextTick 是全局 Vue.nextTick 的别名，其用法相同。
```js
vm.$nextTick( [callback] )
```
+ 参数
  + {Function} [callback]
+ 用法
  将回调延迟到下次 DOM 更新循环之后执行。在修改数据之后立即使用它，然后等待 DOM 更新。它跟全局方法 Vue.nextTick 一样，不同的是回调的 this 自动绑定到调用它的实例上。
```js
<template>
	<div id="example">{{message}}</div>
</template>
<script>
    var vm = new Vue({
      el: '##example',
      data: {
        message: '123'
      }
    })
    vm.message = 'new message' // 更改数据
    console.log(vm.$el.innerHTML) // '123'
    Vue.nextTick(function () {
      console.log(vm.$el.innerHTML) // 'new message'
    })
</script>
```
+ 在上面例子中，当我们更新了message的数据后，立即获取vm.$el.innerHTML，发现此时获取到的还是更新之前的数据：123。但是当我们使用nextTick来获取vm.$el.innerHTML时，此时就可以获取到更新后的数据了。这是为什么呢？
+ 这里就涉及到Vue中对DOM的更新策略了，Vue 在更新 DOM 时是**异步执行**的。只要侦听到数据变化，Vue 将开启一个事件队列，并缓冲在同一事件循环中发生的所有数据变更。如果同一个 watcher 被多次触发，只会被推入到事件队列中一次。这种在缓冲时去除重复数据对于避免不必要的计算和 DOM 操作是非常重要的。然后，在下一个的事件循环“tick”中，Vue 刷新事件队列并执行实际 (已去重的) 工作。
+ 在上面这个例子中，当我们通过 vm.message = 'new message'更新数据时，此时该组件不会立即重新渲染。当刷新事件队列时，组件会在下一个事件循环“tick”中重新渲染。所以当我们更新完数据后，此时又想基于更新后的 DOM 状态来做点什么，此时我们就需要使用Vue.nextTick(callback)，把基于更新后的DOM 状态所需要的操作放入回调函数callback中，这样回调函数将在 DOM 更新完成后被调用。
+ Vue为什么要这么设计？为什么要异步更新DOM？这就涉及到另外一个知识：JS的运行机制。

### JS的运行机制
我们知道 JS 执行是单线程的，它是基于事件循环的。事件循环大致分为以下几个步骤：

1. 所有同步任务都在主线程上执行，形成一个执行栈（execution context stack）。
   
2. 主线程之外，还存在一个"任务队列"（task queue）。只要异步任务有了运行结果，就在"任务队列"之中放置一个事件。
3. 一旦"执行栈"中的所有同步任务执行完毕，系统就会读取"任务队列"，看看里面有哪些事件。那些对应的异步任务，于是结束等待状态，进入执行栈，开始执行。
4. 主线程不断重复上面的第三步。

主线程的执行过程就是一个 tick，而所有的异步结果都是通过 “任务队列” 来调度。 任务队列中存放的是一个个的任务（task）。 规范中规定 task 分为两大类，分别是宏任务(macro task) 和微任务(micro task），并且每执行完一个个宏任务(macro task)后，都要去清空该宏任务所对应的微任务队列中所有的微任务(micro task），他们的执行顺序如下所示：
```js
for (macroTask of macroTaskQueue) {
    // 1. 处理当前的宏任务
    handleMacroTask();

    // 2. 处理对应的所有微任务
    for (microTask of microTaskQueue) {
        handleMicroTask(microTask);
    }
}
```
在浏览器环境中，常见的
+ 宏任务(macro task) 有 setTimeout、MessageChannel、postMessage、setImmediate；
+ 微任务(micro task）有MutationObsever 和 Promise.then。

### 内部原理
nextTick 的定义位于源码的src/core/util/next-tick.js中，其大概可分为两大部分：
+ 能力检测
+ 根据能力检测以不同方式执行回调队列

#### 能力检测
Vue 在内部对异步队列尝试使用原生的 Promise.then、MutationObserver 和 setImmediate，如果执行环境不支持，则会采用 setTimeout(fn, 0) 代替。
宏任务耗费的时间是大于微任务的，所以在浏览器支持的情况下，优先使用微任务。如果浏览器不支持微任务，使用宏任务；但是，各种宏任务之间也有效率的不同，需要根据浏览器的支持情况，使用不同的宏任务。


这一部分的源码如下：
```js
let microTimerFunc
let macroTimerFunc
let useMacroTask = false

/* 对于宏任务(macro task) */
// 检测是否支持原生 setImmediate(高版本 IE 和 Edge 支持)
if (typeof setImmediate !== 'undefined' && isNative(setImmediate)) {
    macroTimerFunc = () => {
        setImmediate(flushCallbacks)
    }
}
// 检测是否支持原生的 MessageChannel
else if (typeof MessageChannel !== 'undefined' && (
    isNative(MessageChannel) ||
    // PhantomJS
    MessageChannel.toString() === '[object MessageChannelConstructor]'
)) {
    const channel = new MessageChannel()
    const port = channel.port2
    channel.port1.onmessage = flushCallbacks
    macroTimerFunc = () => {
        port.postMessage(1)
    }
}
// 都不支持的情况下，使用setTimeout
else {
    macroTimerFunc = () => {
        setTimeout(flushCallbacks, 0)
    }
}

/* 对于微任务(micro task) */
// 检测浏览器是否原生支持 Promise
if (typeof Promise !== 'undefined' && isNative(Promise)) {
    const p = Promise.resolve()
    microTimerFunc = () => {
        p.then(flushCallbacks)
    }
}
// 不支持的话直接指向 macro task 的实现。
else {
    // fallback to macro
    microTimerFunc = macroTimerFunc
}
```
####  执行回调队列

接下来就进入了核心函数nextTick中，如下：
```js
const callbacks = []   // 回调队列
let pending = false    // 异步锁

// 执行队列中的每一个回调
function flushCallbacks () {
    pending = false     // 重置异步锁
    // 防止出现nextTick中包含nextTick时出现问题，在执行回调函数队列前，提前复制备份并清空回调函数队列
    const copies = callbacks.slice(0)
    callbacks.length = 0
    // 执行回调函数队列
    for (let i = 0; i < copies.length; i++) {
        copies[i]()
    }
}

export function nextTick (cb?: Function, ctx?: Object) {
    let _resolve
    // 将回调函数推入回调队列
    callbacks.push(() => {
        if (cb) {
            try {
                cb.call(ctx)
            } catch (e) {
                handleError(e, ctx, 'nextTick')
            }
        } else if (_resolve) {
            _resolve(ctx)
        }
    })
    // 如果异步锁未锁上，锁上异步锁，调用异步函数，准备等同步函数执行完后，就开始执行回调函数队列
    if (!pending) {
        pending = true
        if (useMacroTask) {
            macroTimerFunc()
        } else {
            microTimerFunc()
        }
    }
    // 如果没有提供回调，并且支持Promise，返回一个Promise
    if (!cb && typeof Promise !== 'undefined') {
        return new Promise(resolve => {
            _resolve = resolve
        })
    }
}
```

首先，先来看 nextTick函数，该函数的主要逻辑是：先把传入的回调函数 cb 推入 回调队列callbacks 数组，同时在接收第一个回调函数时，执行能力检测中对应的异步方法（异步方法中调用了回调函数队列）。最后一次性地根据 useMacroTask 条件执行 macroTimerFunc 或者是 microTimerFunc，而它们都会在下一个 tick 执行 flushCallbacks，对 callbacks 遍历，然后执行相应的回调函数
这是当 nextTick 不传 cb 参数的时候，提供一个 Promise 化的调用，比如：
```js
nextTick().then(() => {})
```
当 _resolve 函数执行，就会跳到 then 的逻辑中。

这里有两个问题需要注意：

1. 如何保证只在接收第一个回调函数时执行异步方法？
nextTick源码中使用了一个异步锁的概念，即接收第一个回调函数时，先关上锁，执行异步方法。此时，浏览器处于等待执行完同步代码就执行异步代码的情况。

2. 执行 flushCallbacks 函数时为什么需要备份回调函数队列？执行的也是备份的回调函数队列？
因为，会出现这么一种情况：nextTick 的回调函数中还使用 nextTick。如果 flushCallbacks 不做特殊处理，直接循环执行回调函数，会导致里面nextTick 中的回调函数会进入回调队列。

以上就是对 nextTick 的源码分析，我们了解到数据的变化到 DOM 的重新渲染是一个异步过程，发生在下一个 tick。当我们在实际开发中，比如从服务端接口去获取数据的时候，数据做了修改，如果我们的某些方法去依赖了数据修改后的 DOM 变化，我们就必须在 nextTick 后执行。



## vm.$destory

```js
vm.$destroy()
```
完全销毁一个实例。清理它与其它实例的连接，解绑它的全部指令及事件监听器。
