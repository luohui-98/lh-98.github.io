---
title: vue3学习
tag: vue3
date: 2022-1-5
---

# Vue3与Vue2在应用中的区别

* 响应式数据在Vue3中变得更加灵活和友善。Vue2中 data 里没有定义的属性在后续无法正常的进行响应操作，必须通过 Vue.set 这个 API 向响应式对象中添加一个 property，并确保这个新 property 同样是响应式的，且触发视图更新。它必须用于向响应式对象上添加新 property，因为 Vue 无法探测普通的新增 property (比如 this.myObject.newProperty = 'hi'); 然而在 Vue3 中我们可以通过引入 ref 来操作响应值。ref 是一个实例方法，接受一个内部值并返回一个响应式且可变的 ref 对象。ref 对象具有指向内部值的单个 property.value。
```js
const count = ref(0)
console.log(count.value) // 0

count.value++
console.log(count.value) // 1
```
Vue3 采用了 ES6的一项新特性：Proxy 来实现Vue3中数据响应式的设计。通过下面的伪代码我们可以对比一下：
```js
Object.definProperty(data,'count',{
  get(){},
  set(){}
})
new Proxy(data,'count',{
  get(key){},
  set(key,value){}
})
```
> Object.defineProperty 要修改 data 中的属性必须要明确的知道 key 值（count）, Proxy 在使用中是读取或者设置data中任意的 key，所以不管是修改已有的属性还是新增属性，都可以实现响应式的要求。

# vue3使用
* 关于生命周期钩子函数
| vue2 | vue3 |
| ---- | ---- |
| beforeCreate() | 	use setup() |
| created() | use setup() |
| beforeMount() | onBeforeMount |
| mounted() | onMounted |
| beforeUpdate() | onBeforeUpdate |
| updated() | onUpdated |
| beforeDestory() | onBeforeUnmount |
| destoryed() | onUnmounted |
| activated() | onActivated |
| deactivated() | onDeactivated |
| errorCaptured() | onErrorCaptured |
|  | onRenderTracked(新增) — DebuggerEvent 调试用 |
|  | 	onRenderTriggered(新增) — DebuggerEvent 调试用 |

> Vue3中的钩子函数都在 setup() 中调用。

* computed，watch 可直接调用
```js
const count = ref(0)
const double = computed(() => {
    return count.value * 2
})
```
watch 接收两个参数，第一个参数是监听的属性，多个属性可传入数组， 第二个参数是一个回调函数，回调函数有两个参数（newVal, oldVal）；当 watch 的第一个参数是一个数组时，newVal 与 oldVal 对应的也是数组形式，一一对应。
```js
// 监听count
watch('count', (newVal, oldVal) => {
    console.log('newVal:', newVal)
    console.log('oldVal:', oldVal)
})
// 监听多个属性值
watch(['count', 'name'], (newVal, oldVal) => {
    console.log('newVal:', newVal) // 数组
    console.log('oldVal:', oldVal) // 数组
})
```
如果是需要监听定义在 reacitive 对象中的单一属性，需要通过函数返回值来进行监听。
```js
watch(() => data.count, (newVal, oldVal) => {
    console.log('newVal:', newVal)
    console.log('oldVal:', oldVal)
})
```

* Option API 与 Composition API
  - vue 2.x 使用的是Option API 构建组件。一个组件的功能需要通过methods，computed，watch，data等属性和方法，共同处理页面逻辑。存在多个业务功能共同使用一个实例化new vue()
  这种构建方式在业务逻辑复杂的大项目中，API比较分散，可能会存在分不清哪个方法对应哪个功能。项目的易读性、可复用性相对较差，耦合性较高。
  - vue 3.x 使用的是Composition API 构建组件。代码是根据逻辑功能来组织的，一个功能所定义的所有api会放在一起 （高内聚，低耦合），我们能快速的定位到这个功能所用到的所有API，提高代码可读性和可维护性

* setup函数是使用Composition API的入口
  * 在创建组件实例时，在初始组件解析之后调用setup。在生命周期方面，它在beforeCreate钩子之前调用；
  * 可以返回一个对象，这个对象的属性被合并到渲染上下文，并可以在模板中直接使用
  * 可以返回一个渲染函数，如下： return () => h(‘div’, [count.value, object.foo])
  * 接收props对象作为第一个参数，接收来的props对象，props对象是响应式的(reactive), 当传入的新的props对象时会对其进行更新，且可以通过watchEffect或watch监视其变化。
  props对象不支持解构,解构会导致失去响应性：
  ```js
  export default {
    props: {
      name: String
    },
    setup({ name }) {
      watchEffect(() => {
        console.log(`name is: ` + name) // Will not be reactive!
      })
    }
  }
  ```

  * 接受context对象作为第二个参数，这个对象包含attrs（属性），slots（作用域插槽），emit（事件传播函数）三个属性。（还有expose 函数，实际为4个属性， 可以通过expose 向父级暴露一些子组件的函数、属性等，父组件可以通过ref直接获取到）
  与 prop 不同，context 是普通对象，不是响应式的，slots 和 attr 的值会在组件更新时而更新，如果需要监听 slots 、‘attr’ 的更新触发的副作用，建议在 setup() 函数中添加 onUpdated 函数监听副作用

  ```js
  // comp-a.vue
  {
    name: 'comp-a',
    setup(props, { attrs, slots, emit, expose }) {
      const observed = reactive({
        a: 1
      })
      function setObservedA(value) {
        observed.a = value
      }
      expose({
        setObservedA
      })
      return {
        observed,
      }
    }
  }
  // comp-b.vue
  {
    template: `
      <comp-a ref="compa" />
    `,
    setup() {
      const compa = ref(null)
      onMounted(() => {
        // comp-a 调用 expose 之后, 父组件 ref 拿到的结果为调用 expose 时的参数。而不再是组件实例了
        compa.value.setObservedA(2)
      })
      return {
        compa
      }
    }
  }
  ```
  * setup() 中的 this 不是当前组件实例，实际打印发现为 undefined ， 不建议 setup() 与 Option API 混用，可能会造成混乱。
  
# vue3 中的h函数
* h函数就是vue中的createElement方法，这个函数作用就是创建虚拟dom对象，通过diff算法，追踪dom变化的
* createElement函数，它返回的实际上不是一个DOM元素，更准确的名字是：createNodeDescription（直译为——创建节点描述），因为它所包含的信息会告诉vue页面上需要渲染什么样的节点，包括其子节点的描述信息。我们把这样的节点叫做：“虚拟节点（virtual node）”，也常简写为：“VNode”
* h函数接受三个参数：
  参数一：tag（标签名）、组件的选项对象、函数（必选）；
  参数二：一个对象，标签的属性对应的数据，如：class、id、disabled 等等（可选）；
  参数三：子级虚拟节点，字符串形式或数组形式，子级虚拟节点也需要使用createElement构建。
* dom节点 bable编译前后对比：
![Image text](/imgs/vue3-1.png)

# Vue3.0 toRaw函数和markRaw函数
* toRaw方法是把被reactive或readonly后的Proxy对象转换为原来的target对象，而markRaw则直接让target不能被reactive或readonly

