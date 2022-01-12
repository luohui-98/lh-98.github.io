---
title: vue生命周期-初始化阶段
tag: vue2.0
date: 2021-7-26
---
# new Vue()
初始化阶段所做的第一件事就是new Vue()创建一个Vue实例，那么new Vue()的内部都干了什么呢？ 我们知道，new 关键字在 JS中表示从一个类中实例化出一个对象来，由此可见， Vue 实际上是一个类。所以new Vue()实际上是执行了Vue类的构造函数

## 做了什么
```js
function Vue (options) {
  if (process.env.NODE_ENV !== 'production' &&
    !(this instanceof Vue)
  ) {
    warn('Vue is a constructor and should be called with the `new` keyword')
  }
  this._init(options)
}
```
可以看到，Vue类的定义非常简单，其构造函数核心就一行代码：``` this._init(options) ```

调用原型上的_init(options)方法并把用户所写的选项options传入。那这个_init方法是从哪来的呢？
```js
initMixin(Vue)
```
这一行代码执行了initMixin函数，那initMixin函数又是从哪儿来的呢？该函数定义位于源码的src/core/instance/init.js 中，如下：
```js
export function initMixin (Vue) {
  Vue.prototype._init = function (options) {
    const vm = this
    vm.$options = mergeOptions(
        resolveConstructorOptions(vm.constructor),
        options || {},
        vm
    )
    vm._self = vm
    initLifecycle(vm)
    initEvents(vm)
    initRender(vm)
    callHook(vm, 'beforeCreate')
    initInjections(vm) // resolve injections before data/props
    initState(vm)
    initProvide(vm) // resolve provide after data/props
    callHook(vm, 'created')

    if (vm.$options.el) {
      vm.$mount(vm.$options.el)
    }
  }
}
```
可以看到，在initMixin函数内部就只干了一件事，那就是给Vue类的原型上绑定_init方法，同时_init方法的定义也在该函数内部。现在我们知道了，new Vue()会执行Vue类的构造函数，构造函数内部会执行_init方法，所以new Vue()所干的事情其实就是_init方法所干的事情，那么我们着重来分析下_init方法都干了哪些事情。

首先，把Vue实例赋值给变量vm，并且把用户传递的options选项与当前构造函数的options属性及其父级构造函数的options属性进行合并（关于属性如何合并的问题下面会介绍），得到一个新的options选项赋值给$options属性，并将$options属性挂载到Vue实例上，如下：
```js
vm.$options = mergeOptions(
    resolveConstructorOptions(vm.constructor),
    options || {},
    vm
)
```
接着，通过调用一些初始化函数来为Vue实例初始化一些属性，事件，响应式数据等，如下：
```js
initLifecycle(vm)       // 初始化生命周期
initEvents(vm)        // 初始化事件
initRender(vm)         // 初始化渲染
callHook(vm, 'beforeCreate')  // 调用生命周期钩子函数
initInjections(vm)   //初始化injections
initState(vm)    // 初始化props,methods,data,computed,watch
initProvide(vm) // 初始化 provide
callHook(vm, 'created')  // 调用生命周期钩子函数
```
可以看到，除了调用初始化函数来进行相关数据的初始化之外，还在合适的时机调用了callHook函数来触发生命周期的钩子，关于callHook函数是如何触发生命周期的钩子会在下面介绍，我们先继续往下看：
```js
if (vm.$options.el) {
    vm.$mount(vm.$options.el)
}
```

## callHook函数如何触发钩子函数
```js
export function callHook (vm: Component, hook: string) {
  const handlers = vm.$options[hook] // 要执行的钩子函数
  if (handlers) {
    for (let i = 0, j = handlers.length; i < j; i++) {
      try {
        handlers[i].call(vm)  // 执行钩子函数中的每一个方法
      } catch (e) {
        handleError(e, vm, `${hook} hook`)
      }
    }
  }
}
```
首先从实例的$options中获取到需要触发的钩子名称所对应的钩子函数数组handlers，每个生命周期钩子名称都对应了一个钩子函数数组。然后遍历该数组，将数组中的每个钩子函数都执行一遍。

##  initLifecycle函数

```js
export function initLifecycle (vm: Component) {
  const options = vm.$options

  let parent = options.parent
  if (parent && !options.abstract) {
    while (parent.$options.abstract && parent.$parent) {
      parent = parent.$parent
    }
    parent.$children.push(vm)
  }

  vm.$parent = parent
  vm.$root = parent ? parent.$root : vm

  vm.$children = []
  vm.$refs = {}

  vm._watcher = null
  vm._inactive = null
  vm._directInactive = false
  vm._isMounted = false
  vm._isDestroyed = false
  vm._isBeingDestroyed = false
}
```
主要是给Vue实例上挂载了一些属性并设置了默认值，**同时挂载$parent 属性和$root属性**

首先是给实例上挂载$parent属性：
```js
let parent = options.parent
if (parent && !options.abstract) {
  while (parent.$options.abstract && parent.$parent) {
    parent = parent.$parent
  }
  parent.$children.push(vm)
}

vm.$parent = parent
```
如果当前组件不是抽象组件并且存在父级，那么就通过while循环来向上循环，如果当前组件的父级是抽象组件并且也存在父级，那就继续向上查找当前组件父级的父级，直到找到第一个不是抽象类型的父级时，将其赋值vm.$parent，同时把该实例自身添加进找到的父级的$children属性中。这样就确保了在子组件的$parent属性上能访问到父组件实例，在父组件的$children属性上也能访问子组件的实例


接着是给实例上挂载$root属性
```js
vm.$root = parent ? parent.$root : vm
```
实例的$root属性表示当前实例的根实例，挂载该属性时，首先会判断如果当前实例存在父级，那么当前实例的根实例$root属性就是其父级的根实例$root属性，如果不存在，那么根实例$root属性就是它自己。这很好理解，举个例子：假如有一个人，他如果有父亲，那么他父亲的祖先肯定也是他的祖先，同理，他的儿子的祖先也肯定是他的祖先，我们不需要真正的一层一层的向上递归查找到他祖先本人，只需要知道他父亲的祖先是谁然后告诉他即可。如果他没有父亲，那说明他自己就是祖先，那么他后面的儿子、孙子的$root属性就是他自己了

## 解析事件 initEvents
在Vue中，当我们在父组件中使用子组件时可以给子组件上注册一些事件，这些事件即包括使用v-on或@注册的自定义事件，也包括注册的浏览器原生事件（需要加 .native 修饰符），如下：
```html
<child @select="selectHandler" 	@click.native="clickHandler"></child>
```

模板编译解析中，当遇到开始标签的时候，除了会解析开始标签，还会调用processAttrs 方法解析标签中的属性，processAttrs 方法位于源码的 src/compiler/parser/index.js中， 如下：
```js
export const onRE = /^@|^v-on:/
export const dirRE = /^v-|^@|^:/

function processAttrs (el) {
  const list = el.attrsList
  let i, l, name, value, modifiers
  for (i = 0, l = list.length; i < l; i++) {
    name  = list[i].name
    value = list[i].value
    if (dirRE.test(name)) {
      // 解析修饰符
      modifiers = parseModifiers(name)
      if (modifiers) {
        name = name.replace(modifierRE, '')
      }
      if (onRE.test(name)) { // v-on
        name = name.replace(onRE, '')
        addHandler(el, name, value, modifiers, false, warn)
      }
    }
  }
}
```

在对标签属性进行解析时，判断如果属性是指令，首先通过 parseModifiers 解析出属性的修饰符，然后判断如果是事件的指令，则执行 addHandler(el, name, value, modifiers, false, warn) 方法， 该方法定义在 src/compiler/helpers.js 中，如下：
```js
export function addHandler (el,name,value,modifiers) {
  modifiers = modifiers || emptyObject

  // check capture modifier 判断是否有capture修饰符
  if (modifiers.capture) {
    delete modifiers.capture
    name = '!' + name // 给事件名前加'!'用以标记capture修饰符
  }
  // 判断是否有once修饰符
  if (modifiers.once) {
    delete modifiers.once
    name = '~' + name // 给事件名前加'~'用以标记once修饰符
  }
  // 判断是否有passive修饰符
  if (modifiers.passive) {
    delete modifiers.passive
    name = '&' + name // 给事件名前加'&'用以标记passive修饰符
  }

  let events
  if (modifiers.native) {
    delete modifiers.native
    events = el.nativeEvents || (el.nativeEvents = {})
  } else {
    events = el.events || (el.events = {})
  }

  const newHandler: any = {
    value: value.trim()
  }
  if (modifiers !== emptyObject) {
    newHandler.modifiers = modifiers
  }

  const handlers = events[name]
  if (Array.isArray(handlers)) {
    handlers.push(newHandler)
  } else if (handlers) {
    events[name] = [handlers, newHandler]
  } else {
    events[name] = newHandler
  }

  el.plain = false
}
```

在addHandler 函数里做了 3 件事情，首先根据 modifier 修饰符对事件名 name 做处理，接着根据 modifier.native 判断事件是一个浏览器原生事件还是自定义事件，分别对应 el.nativeEvents 和 el.events，最后按照 name 对事件做归类，并把回调函数的字符串保留到对应的事件中。


##  initInjections

从函数名字上来看，该函数是用来初始化实例中的inject选项的。说到inject选项，那必然离不开provide选项，这两个选项都是成对出现的，它们的作用是：允许一个祖先组件向其所有子孙后代注入一个依赖，不论组件层次有多深，并在起上下游关系成立的时间里始终生效。

```js
export function initInjections (vm: Component) {
  const result = resolveInject(vm.$options.inject, vm)
  if (result) {
    toggleObserving(false)
    Object.keys(result).forEach(key => {
      defineReactive(vm, key, result[key])
    }
    toggleObserving(true)
  }
}

export let shouldObserve: boolean = true
export function toggleObserving (value: boolean) {
  shouldObserve = value
}
```
首先调用resolveInject把inject选项中的数据转化成键值对的形式赋给result，如官方文档给出的例子，那么result应为如下样子：
```js
// 父级组件提供 'foo'
var Parent = {
  provide: {
    foo: 'bar'
  }
}

// 子组件注入 'foo'
var Child = {
  inject: ['foo'],
}

// result
result = {
    'foo':'bar'
}
```
然后遍历result中的每一对键值，调用defineReactive函数将其添加当前实例上，如下：
```js
if (result) {
    toggleObserving(false)
    Object.keys(result).forEach(key => {
        defineReactive(vm, key, result[key])
    }
    toggleObserving(true)
}
```
此处有一个地方需要注意，在把result中的键值添加到当前实例上之前，会先调用toggleObserving(false)，而这个函数内部是把shouldObserve = false，这是为了告诉defineReactive函数仅仅是把键值添加到当前实例上而不需要将其转换成响应式，这个就呼应了官方文档在介绍provide 和 inject 选项用法的时候所提示的：

**provide 和 inject 绑定并不是可响应的。这是刻意为之的。然而，如果你传入了一个可监听的对象，那么其对象的属性还是可响应的。**

resolveInject
```js
export function resolveInject (inject: any, vm: Component): ?Object {
  if (inject) {
    const result = Object.create(null)
    const keys =  Object.keys(inject)

    for (let i = 0; i < keys.length; i++) {
      const key = keys[i]
      const provideKey = inject[key].from
      let source = vm
      while (source) {
        if (source._provided && hasOwn(source._provided, provideKey)) {
          result[key] = source._provided[provideKey]
          break
        }
        source = source.$parent
      }
      if (!source) {
        if ('default' in inject[key]) {
          const provideDefault = inject[key].default
          result[key] = typeof provideDefault === 'function'
            ? provideDefault.call(vm)
            : provideDefault
        } else if (process.env.NODE_ENV !== 'production') {
          warn(`Injection "${key}" not found`, vm)
        }
      }
    }
    return result
  }
}
```

## initState 初始化实例状态

从函数名字上来看，这个函数是用来初始化实例状态的,在Vue组件中会写一些如**props、data、methods、computed、watch**选项，我们把这些选项称为实例的状态选项。也就是说，initState函数就是用来初始化这些状态的。

```js
export function initState (vm: Component) {
  vm._watchers = []
  const opts = vm.$options
  if (opts.props) initProps(vm, opts.props)
  if (opts.methods) initMethods(vm, opts.methods)
  if (opts.data) {
    initData(vm)
  } else {
    observe(vm._data = {}, true /* asRootData */)
  }
  if (opts.computed) initComputed(vm, opts.computed)
  if (opts.watch && opts.watch !== nativeWatch) {
    initWatch(vm, opts.watch)
  }
}
```

首先，给实例上新增了一个属性_watchers，用来存储当前实例中所有的watcher实例，无论是使用vm.$watch注册的watcher实例还是使用watch选项注册的watcher实例，都会被保存到该属性中。

这里我们再额外多说一点，在变化侦测篇中我们介绍了Vue中对数据变化的侦测是使用属性拦截的方式实现的，但是Vue并不是对所有数据都使用属性拦截的方式侦测变化，这是因为数据越多，数据上所绑定的依赖就会多，从而造成依赖追踪的内存开销就会很大，所以从Vue 2.0版本起，Vue不再对所有数据都进行侦测，而是将侦测粒度提高到了组件层面，对每个组件进行侦测，所以在每个组件上新增了vm._watchers属性，用来存放这个组件内用到的所有状态的依赖，当其中一个状态发生变化时，就会通知到组件，然后由组件内部使用虚拟DOM进行数据比对，从而降低内存开销，提高性能。
+ 先判断实例中是否有props选项，如果有，就调用props选项初始化函数initProps去初始化props选项；
+ 再判断实例中是否有methods选项，如果有，就调用methods选项初始化函数initMethods去初始化methods选项；
+ 接着再判断实例中是否有data选项，如果有，就调用data选项初始化函数initData去初始化data选项；如果没有，就把data当作空对象并将其转换成响应式；
+ 接着再判断实例中是否有computed选项，如果有，就调用computed选项初始化函数initComputed去初始化computed选项；
+ 最后判断实例中是否有watch选项，如果有，就调用watch选项初始化函数initWatch去初始化watch选项；
+ 总之一句话就是：有什么选项就调用对应的选项初始化子函数去初始化什么选项。

### 初始化props
props选项通常是由当前组件的父级组件传入的，当父组件在调用子组件的时候，通常会把props属性值作为标签属性添加在子组件的标签上，如下：
```html
<Child prop1="xxx" prop2="yyy"></Child>
```
>在模板编译的时候，当解析到组件标签时会将所有的标签属性都解析出来然后在子组件实例化的时候传给子组件，当然这里面就包括props数据。

子组件接受：
```js
// 写法一
props: ['name']
// 写法二
props: {
    name: String, // [String, Number]
}
// 写法三
props: {
    name:{
		type: String
    }
}
```
>Vue给用户提供的props选项写法非常自由，写法虽多但是最终处理的时候肯定只处理一种写法，将所有写法都转化成一种写法。
```js
function normalizeProps (options, vm) {
  const props = options.props
  if (!props) return
  const res = {}
  let i, val, name
  if (Array.isArray(props)) {
    i = props.length
    while (i--) {
      val = props[i]
      if (typeof val === 'string') {
        name = camelize(val)
        res[name] = { type: null }
      } else if (process.env.NODE_ENV !== 'production') {
        warn('props must be strings when using array syntax.')
      }
    }
  } else if (isPlainObject(props)) {
    for (const key in props) {
      val = props[key]
      name = camelize(key)
      res[name] = isPlainObject(val)
        ? val
        : { type: val }
    }
  } else if (process.env.NODE_ENV !== 'production') {
    warn(
      `Invalid value for option "props": expected an Array or an Object, ` +
      `but got ${toRawType(props)}.`,
      vm
    )
  }
  options.props = res
}
```
上面代码中，首先拿到实例中的props选项，如果不存在，则直接返回。

如果存在，则定义一个空对象res，用来存储最终的结果。接着判断如果props选项是一个数组（写法一），则遍历该数组中的每一项元素，如果该元素是字符串，那么先将该元素统一转化成驼峰式命名，然后将该元素作为key，将{type: null}作为value存入res中；如果该元素不是字符串，则抛出异常。

如果props选项不是数组那就继续判断是不是一个对象，如果是一个对象，那就遍历对象中的每一对键值，拿到每一对键值后，先将键名统一转化成驼峰式命名，然后判断值是否还是一个对象，如果值是对象（写法三），那么就将该键值对存入res中；如果值不是对象（写法二），那么就将键名作为key，将{type: null}作为value存入res中。

如果props选项既不是数组也不是对象，那么如果在非生产环境下就抛出异常，最后将res作为规范化后的结果重新赋值给实例的props选项。

无论是三种写法的哪一种，最终都会被转化成如下写法：

```js
props: {
    name:{
        type: xxx
    }
}
```
####  initProps函数分析
将props选项规范化完成之后，接下来我们就可以来真正的初始化props选项了，initProps函数的定义位于源码的src/core/instance/state.js中，如下：
```js
function initProps (vm: Component, propsOptions: Object) {
  const propsData = vm.$options.propsData || {}
  const props = vm._props = {}
  // cache prop keys so that future props updates can iterate using Array
  // instead of dynamic object key enumeration.
  const keys = vm.$options._propKeys = []
  const isRoot = !vm.$parent
  // root instance props should be converted
  if (!isRoot) {
    toggleObserving(false)
  }
  for (const key in propsOptions) {
    keys.push(key)
    const value = validateProp(key, propsOptions, propsData, vm)
    /* istanbul ignore else */
    if (process.env.NODE_ENV !== 'production') {
      const hyphenatedKey = hyphenate(key)
      if (isReservedAttribute(hyphenatedKey) ||
          config.isReservedAttr(hyphenatedKey)) {
        warn(
          `"${hyphenatedKey}" is a reserved attribute and cannot be used as component prop.`,
          vm
        )
      }
      defineReactive(props, key, value, () => {
        if (vm.$parent && !isUpdatingChildComponent) {
          warn(
            `Avoid mutating a prop directly since the value will be ` +
            `overwritten whenever the parent component re-renders. ` +
            `Instead, use a data or computed property based on the prop's ` +
            `value. Prop being mutated: "${key}"`,
            vm
          )
        }
      })
    } else {
      defineReactive(props, key, value)
    }
    // static props are already proxied on the component's prototype
    // during Vue.extend(). We only need to proxy props defined at
    // instantiation here.
    if (!(key in vm)) {
      proxy(vm, `_props`, key)
    }
  }
  toggleObserving(true)
}

```
可以看到，该函数接收两个参数：当前Vue实例和当前实例规范化后的props选项。
在函数内部首先定义了4个变量：
+ propsData:父组件传入的真实props数据。
+ props:指向vm._props的指针，所有设置到props变量中的属性都会保存到vm._props中。
+ keys:指向vm.$options._propKeys的指针，缓存props对象中的key，将来更新props时只需遍历vm.$options._propKeys数组即可得到所有props的key。
+ isRoot:当前组件是否为根组件。

接着，判断当前组件是否为根组件，如果不是，那么不需要将props数组转换为响应式的，toggleObserving(false)用来控制是否将数据转换成响应式。

接着，遍历props选项拿到每一对键值，先将键名添加到keys中，然后调用validateProp函数（关于该函数下面会介绍）校验父组件传入的props数据类型是否匹配并获取到传入的值value，然后将键和值通过defineReactive函数添加到props（即vm._props）中。

添加完之后再判断这个key在当前实例vm中是否存在，如果不存在，则调用proxy函数在vm上设置一个以key为属性的代码，当使用vm[key]访问数据时，其实访问的是vm._props[key]。

#### validateProp函数分析

```js
export function validateProp (key,propOptions,propsData,vm) {
  const prop = propOptions[key]
  const absent = !hasOwn(propsData, key)
  let value = propsData[key]
  // boolean casting
  const booleanIndex = getTypeIndex(Boolean, prop.type)
  if (booleanIndex > -1) {
    if (absent && !hasOwn(prop, 'default')) {
      value = false
    } else if (value === '' || value === hyphenate(key)) {
      // only cast empty string / same name to boolean if
      // boolean has higher priority
      const stringIndex = getTypeIndex(String, prop.type)
      if (stringIndex < 0 || booleanIndex < stringIndex) {
        value = true
      }
    }
  }
  // check default value
  if (value === undefined) {
    value = getPropDefaultValue(vm, prop, key)
    // since the default value is a fresh copy,
    // make sure to observe it.
    const prevShouldObserve = shouldObserve
    toggleObserving(true)
    observe(value)
    toggleObserving(prevShouldObserve)
  }
  if (process.env.NODE_ENV !== 'production') {
    assertProp(prop, key, value, vm, absent)
  }
  return value
}
```
可以看到，该函数接收4个参数，分别是：

+ key:遍历propOptions时拿到的每个属性名。
+ propOptions:当前实例规范化后的props选项。
+ propsData:父组件传入的真实props数据。
+ vm:当前实例。

在函数内部首先定义了3个变量，分别是：
+ prop:当前key在propOptions中对应的值。
+ absent:当前key是否在propsData中存在，即父组件是否传入了该属性。
+ value:当前key在propsData中对应的值，即父组件对于该属性传入的真实值。

接着，判断prop的type属性是否是布尔类型（Boolean）,getTypeIndex函数用于判断prop的type属性中是否存在某种类型，如果存在，则返回该类型在type属性中的索引（因为type属性可以是数组），如果不存在则返回-1。

如果是布尔类型的话，那么有两种边界情况需要单独处理：

1. 如果absent为true，即父组件没有传入该prop属性并且该属性也没有默认值的时候，将该属性值设置为false，如下：
```js
   if (absent && !hasOwn(prop, 'default')) {
    value = false
  }
```
2. 如果父组件传入了该prop属性，那么需要满足以下几点：
  + 该属性值为空字符串或者属性值与属性名相等；
  + prop的type属性中不存在String类型；
  + 如果prop的type属性中存在String类型，那么Boolean类型在type属性中的索引必须小于String类型的索引，即Boolean类型的优先级更高;
  则将该属性值设置为true，如下：
```js
if (value === '' || value === hyphenate(key)) {
    const stringIndex = getTypeIndex(String, prop.type)
    if (stringIndex < 0 || booleanIndex < stringIndex) {
        value = true
    }
}
```
另外，在判断属性值与属性名相等的时候，是先将属性名由驼峰式转换成用-连接的字符串，下面的这几种写法，子组件的prop都将被设置为true：

```html
<Child name></Child>
<Child name="name"></Child>
<Child userName="user-name"></Child>
```
如果不是布尔类型，是其它类型的话，那就只需判断父组件是否传入该属性即可，如果没有传入，则该属性值为undefined，此时调用getPropDefaultValue函数（关于该函数下面会介绍）获取该属性的默认值，并将其转换成响应式，如下：
```js
if (value === undefined) {
    value = getPropDefaultValue(vm, prop, key)
    // since the default value is a fresh copy,
    // make sure to observe it.
    const prevShouldObserve = shouldObserve
    toggleObserving(true)
    observe(value)
    toggleObserving(prevShouldObserve)
}
```
如果父组件传入了该属性并且也有对应的真实值，那么在非生产环境下会调用assertProp函数（关于该函数下面会介绍）校验该属性值是否与要求的类型相匹配。如下：
```js
if (process.env.NODE_ENV !== 'production' ) {
    assertProp(prop, key, value, vm, absent)
}
```
最后将父组件传入的该属性的真实值返回。

### 初始化methods
它的初始化函数定义位于源码的src/core/instance/state.js中，如下：
```js
function initMethods (vm, methods) {
  const props = vm.$options.props
  for (const key in methods) {
    if (process.env.NODE_ENV !== 'production') {
      if (methods[key] == null) {
        warn(
          `Method "${key}" has an undefined value in the component definition. ` +
          `Did you reference the function correctly?`,
          vm
        )
      }
      if (props && hasOwn(props, key)) {
        warn(
          `Method "${key}" has already been defined as a prop.`,
          vm
        )
      }
      if ((key in vm) && isReserved(key)) {
        warn(
          `Method "${key}" conflicts with an existing Vue instance method. ` +
          `Avoid defining component methods that start with _ or $.`
        )
      }
    }
    vm[key] = methods[key] == null ? noop : bind(methods[key], vm)
  }
}
```
初始化methods无非就干了三件事：判断method有没有？method的命名符不符合命名规范？如果method既有又符合规范那就把它挂载到vm实例上。下面我们就逐行分析源码，来过一遍这三件事。

首先，遍历methods选项中的每一个对象，在非生产环境下判断如果methods中某个方法只有key而没有value，即只有方法名没有方法体时，抛出异常：提示用户方法未定义。
接着判断如果methods中某个方法名与props中某个属性名重复了，就抛出异常：提示用户方法名重复了。
接着判断如果methods中某个方法名如果在实例vm中已经存在并且方法名是以_或$开头的，就抛出异常：提示用户方法名命名不规范。（其中，isReserved函数是用来判断字符串是否以_或$开头）
最后，如果上述判断都没问题，那就method绑定到实例vm上，这样，我们就可以通过this.xxx来访问methods选项中的xxx方法了，如下：
```js
vm[key] = methods[key] == null ? noop : bind(methods[key], vm)
```

### 初始化data
它的初始化函数定义位于源码的src/core/instance/state.js中，如下：
```js
function initData (vm) {
    let data = vm.$options.data
    data = vm._data = typeof data === 'function'
        ? getData(data, vm)
    : data || {}
    if (!isPlainObject(data)) {
        data = {}
        process.env.NODE_ENV !== 'production' && warn(
            'data functions should return an object:\n' +
            'https://vuejs.org/v2/guide/components.html##data-Must-Be-a-Function',
            vm
        )
    }
    // proxy data on instance
    const keys = Object.keys(data)
    const props = vm.$options.props
    const methods = vm.$options.methods
    let i = keys.length
    while (i--) {
        const key = keys[i]
        if (process.env.NODE_ENV !== 'production') {
            if (methods && hasOwn(methods, key)) {
                warn(
                    `Method "${key}" has already been defined as a data property.`,
                    vm
                )
            }
        }
        if (props && hasOwn(props, key)) {
            process.env.NODE_ENV !== 'production' && warn(
                `The data property "${key}" is already declared as a prop. ` +
                `Use prop default value instead.`,
                vm
            )
        } else if (!isReserved(key)) {
            proxy(vm, `_data`, key)
        }
    }
    // observe data
    observe(data, true /* asRootData */)
}
```
跟initMethods函数的逻辑有几分相似。就是通过一系列条件判断用户传入的data选项是否合法，最后将data转换成响应式并绑定到实例vm上。下面我们就来仔细看一下代码逻辑。

首先获取到用户传入的data选项，赋给变量data，同时将变量data作为指针指向vm._data，然后判断data是不是一个函数，如果是就调用getData函数获取其返回值，将其保存到vm._data中。如果不是，就将其本身保存到vm._data中。
```js
let data = vm.$options.data
data = vm._data = typeof data === 'function'
    ? getData(data, vm)
	: data || {}
```
我们知道，无论传入的data选项是不是一个函数，它最终的值都应该是一个对象，如果不是对象的话，就抛出警告：提示用户data应该是一个对象。
```js
if (!isPlainObject(data)) {
    data = {}
    process.env.NODE_ENV !== 'production' && warn(
        'data functions should return an object:\n' +
        'https://vuejs.org/v2/guide/components.html##data-Must-Be-a-Function',
        vm
    )
}
```
接下来遍历data对象中的每一项，在非生产环境下判断data对象中是否存在某一项的key与methods中某个属性名重复，如果存在重复，就抛出警告：提示用户属性名重复。
```
if (process.env.NODE_ENV !== 'production') {
    if (methods && hasOwn(methods, key)) {
        warn(
            `Method "${key}" has already been defined as a data property.`,
            vm
        )
    }
}
```
接着再判断是否存在某一项的key与prop中某个属性名重复，如果存在重复，就抛出警告：提示用户属性名重复。
```
if (props && hasOwn(props, key)) {
    process.env.NODE_ENV !== 'production' && warn(
        `The data property "${key}" is already declared as a prop. ` +
        `Use prop default value instead.`,
        vm
    )
}
```
如果都没有重复，则调用proxy函数将data对象中key不以_或$开头的属性代理到实例vm上，这样，我们就可以通过this.xxx来访问data选项中的xxx数据了。

最后，调用observe函数将data中的数据转化成响应式.
```js
observe(data, true /* asRootData */)
```
### 初始化computed

计算属性computed相信大家一定不会陌生，在日常开发中肯定会经常用到，而且我们知道计算属性有一个很大的特点就是： 计算属性的结果会被缓存，除非依赖的响应式属性变化才会重新计算。 
首先，根据官方文档的使用示例，我们来回顾一下计算属性的用法:
```js
var vm = new Vue({
  data: { a: 1 },
  computed: {
    // 仅读取
    aDouble: function () {
      return this.a * 2
    },
    // 读取和设置
    aPlus: {
      get: function () {
        return this.a + 1
      },
      set: function (v) {
        this.a = v - 1
      }
    }
  }
})
vm.aPlus   // => 2
vm.aPlus = 3
vm.a       // => 2
vm.aDouble // => 4
```
computed选项中的属性值可以是一个函数，那么该函数默认为取值器getter，用于仅读取数据；还可以是一个对象，对象里面有取值器getter和存值器setter，用于读取和设置数据。
 #### initComputed函数分析
initComputed函数的定义位于源码的src/core/instance/state.js中，如下：
```js
function initComputed (vm: Component, computed: Object) {
    const watchers = vm._computedWatchers = Object.create(null)
    const isSSR = isServerRendering()

    for (const key in computed) {
        const userDef = computed[key]
        const getter = typeof userDef === 'function' ? userDef : userDef.get
        if (process.env.NODE_ENV !== 'production' && getter == null) {
            warn(
                `Getter is missing for computed property "${key}".`,
                vm
            )
        }

        if (!isSSR) {
            // create internal watcher for the computed property.
            watchers[key] = new Watcher(
                vm,
                getter || noop,
                noop,
                computedWatcherOptions
            )
        }

        if (!(key in vm)) {
            defineComputed(vm, key, userDef)
        } else if (process.env.NODE_ENV !== 'production') {
            if (key in vm.$data) {
                warn(`The computed property "${key}" is already defined in data.`, vm)
            } else if (vm.$options.props && key in vm.$options.props) {
                warn(`The computed property "${key}" is already defined as a prop.`, vm)
            }
        }
    }
}
```
可以看到，在函数内部，首先定义了一个变量watchers并将其赋值为空对象，同时将其作为指针指向vm._computedWatchers
接着，遍历computed选项中的每一项属性，首先获取到每一项的属性值，记作userDef，然后判断userDef是不是一个函数，如果是函数，则该函数默认为取值器getter，将其赋值给变量getter；如果不是函数，则说明是一个对象，则取对象中的get属性作为取值器赋给变量getter。
接着判断在非生产环境下如果上面两种情况取到的取值器不存在，则抛出警告：提示用户计算属性必须有取值器。
接着判断如果不是在服务端渲染环境下，则创建一个watcher实例，并将当前循环到的的属性名作为键，创建的watcher实例作为值存入watchers对象中。
最后，判断当前循环到的的属性名是否存在于当前实例vm上，如果存在，则在非生产环境下抛出警告；如果不存在，则调用defineComputed函数为实例vm上设置计算属性。
### 初始化watch

根据官方文档的使用示例，我们来回顾一下watch选项的用法:
```js
var vm = new Vue({
  data: {
    a: 1,
    b: 2,
    c: 3,
    d: 4,
    e: {
      f: {
        g: 5
      }
    }
  },
  watch: {
    a: function (val, oldVal) {
      console.log('new: %s, old: %s', val, oldVal)
    },
    // methods选项中的方法名
    b: 'someMethod',
    // 深度侦听，该回调会在任何被侦听的对象的 property 改变时被调用，不论其被嵌套多深
    c: {
      handler: function (val, oldVal) { /* ... */ },
      deep: true
    },
    // 该回调将会在侦听开始之后被立即调用
    d: {
      handler: 'someMethod',
      immediate: true
    },
    // 调用多个回调
    e: [
      'handle1',
      function handle2 (val, oldVal) { /* ... */ },
      {
        handler: function handle3 (val, oldVal) { /* ... */ },
      }
    ],
    // 侦听表达式
    'e.f': function (val, oldVal) { /* ... */ }
  }
})
vm.a = 2 // => new: 2, old: 1
```
watch选项的用法非常灵活。首先watch选项是一个对象，键是需要观察的表达式，值是对应回调函数。值也可以是方法名，或者包含选项的对象。既然给用户提供的用法灵活，那么在代码中就需要按条件来判断，根据不同的用法做相应的处理。
### initWatch函数分析
```js
function initWatch (vm, watch) {
  for (const key in watch) {
    const handler = watch[key]
    if (Array.isArray(handler)) {
      for (let i = 0; i < handler.length; i++) {
        createWatcher(vm, key, handler[i])
      }
    } else {
      createWatcher(vm, key, handler)
    }
  }
}
```
可以看到，在函数内部会遍历watch选项，拿到每一项的key和对应的值handler。然后判断handler是否为数组，如果是数组则循环该数组并将数组中的每一项依次调用createWatcher函数来创建watcher；如果不是数组，则直接调用createWatcher函数来创建watcher。

### createWatcher函数分析

```js
function createWatcher (
  vm: Component,
  expOrFn: string | Function,
  handler: any,
  options?: Object
) {
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
可以看到，该函数接收4个参数，分别是：

+ vm:当前实例；
+ expOrFn:被侦听的属性表达式
+ handler:watch选项中每一项的值
+ options:用于传递给vm.$watch的选项对象

在该函数内部，首先会判断传入的handler是否为一个对象，如果是一个对象，那么就认为用户使用的是这种写法：
```js
watch: {
    c: {
        handler: function (val, oldVal) { /* ... */ },
		deep: true
    }
}
```
即带有侦听选项的写法，此时就将handler对象整体记作options，把handler对象中的handler属性作为真正的回调函数记作handler，如下：
```js
if (isPlainObject(handler)) {
    options = handler
    handler = handler.handler
}
```
接着判断传入的handler是否为一个字符串，如果是一个字符串，那么就认为用户使用的是这种写法：
```js
watch: {
    // methods选项中的方法名
    b: 'someMethod',
}
```
即回调函数是methods选项中的一个方法名，我们知道，在初始化methods选项的时候会将选项中的每一个方法都绑定到当前实例上，所以此时我们只需从当前实例上取出该方法作为真正的回调函数记作handler，如下：
```js
if (typeof handler === 'string') {
    handler = vm[handler]
}
```
如果既不是对象又不是字符串，那么我们就认为它是一个函数，就不做任何处理。
