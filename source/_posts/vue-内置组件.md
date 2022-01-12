---
title: vue内置组件
tag: vue2.0
date: 2021-8-26
---

<keep-alive> 是 Vue 实现的一个内置组件，也就是说 Vue 源码不仅实现了一套组件化的机制，也实现了一些内置组件，关于<keep-alive>组件，官网如下介绍：

<keep-alive>是Vue中内置的一个抽象组件，它自身不会渲染一个 DOM 元素，也不会出现在父组件链中。当它包裹动态组件时，会缓存不活动的组件实例，而不是销毁它们。

这句话的意思简单来说：就是我们可以把一些不常变动的组件或者需要缓存的组件用<keep-alive>包裹起来，这样<keep-alive>就会帮我们把组件保存在内存中，而不是直接的销毁，这样做可以保留组件的状态或避免多次重新渲染，以提高页面性能。
```html
<div id="app">
  <button @click="switchComp('child1')">组件1</button>
  <button @click="switchComp('child2')">组件2</button>
  <component :is="chooseComponent"></component>
</div>
<body>
  <script src="https://cdn.jsdelivr.net/npm/vue/dist/vue.js"></script>
  <script>
    var child1 = {
      template: '<div>组件1:<input type="text"/></div>',
    }
    var child2 = {
      template: '<div>组件2:<input type="text"/></div>'
    }
    var vm = new Vue({
      el: '##app',
      components: {
        child1,
        child2,
      },
      data() {
        return {
          chooseComponent: 'child1',
        }
      },
      methods: {
        switchComp(component) {
          this.chooseComponent = component;
        }
      }
    })
  </script>
</body>
```
可以看到，上述代码中定义了两个子组件child1和child2，然后使用两个按钮和一个动态组件来做出点击按钮切换不同组件的效果
我们给组件1和组件2的输入框中分别输入了不同的内容后，之后当我们点击按钮切换组件的时候，切换之前输入的内容已经不存在了，这就说明点击按钮切换组件是把之前的组件销毁，然后又重新挂载了一次。

但是我们有时候又会有这样的需求：当用户再次切回组件时保留切走之前的组件状态。此时Vue内置的<keep-alive>组件就派上用场了，我们将上述代码中的动态组件用<keep-alive>包裹一下，如下：
```html
<div id="app">
  <button @click="switchComp('child1')">组件1</button>
  <button @click="switchComp('child2')">组件2</button>
  <keep-alive>
    <component :is="chooseComponent"></component>
  </keep-alive>
</div>
```
# 用法
<keep-alive>组件可接收三个属性：

+ include - 字符串或正则表达式。只有名称匹配的组件会被缓存。
+ exclude - 字符串或正则表达式。任何名称匹配的组件都不会被缓存。
+ max - 数字。最多可以缓存多少组件实例。

include 和 exclude 属性允许组件有条件地缓存。二者都可以用逗号分隔字符串、正则表达式或一个数组来表示
```html
<!-- 逗号分隔字符串 -->
<keep-alive include="a,b">
  <component :is="view"></component>
</keep-alive>

<!-- 正则表达式 (使用 `v-bind`) -->
<keep-alive :include="/a|b/">
  <component :is="view"></component>
</keep-alive>

<!-- 数组 (使用 `v-bind`) -->
<keep-alive :include="['a', 'b']">
  <component :is="view"></component>
</keep-alive>

<keep-alive :max="10">
  <component :is="view"></component>
</keep-alive>
```
匹配时首先检查组件自身的 name 选项，如果 name 选项不可用，则匹配它的局部注册名称 (父组件 components 选项的键值)，也就是组件的标签值。匿名组件不能被匹配。

max表示最多可以缓存多少组件实例。一旦这个数字达到了，在新实例被创建之前，**已缓存组件中最久没有被访问的实例**会被销毁掉。

# 实现原理
<keep-alive>组件的定义位于源码的 src/core/components/keep-alive.js 文件中，如下：

```js
export default {
  name: 'keep-alive',
  abstract: true,

  props: {
    include: [String, RegExp, Array],
    exclude: [String, RegExp, Array],
    max: [String, Number]
  },

  created () {
    this.cache = Object.create(null)
    this.keys = []
  },

  destroyed () {
    for (const key in this.cache) {
      pruneCacheEntry(this.cache, key, this.keys)
    }
  },

  mounted () {
    this.$watch('include', val => {
      pruneCache(this, name => matches(val, name))
    })
    this.$watch('exclude', val => {
      pruneCache(this, name => !matches(val, name))
    })
  },

  render() {
    /* 获取默认插槽中的第一个组件节点 */
    const slot = this.$slots.default
    const vnode = getFirstComponentChild(slot)
    /* 获取该组件节点的componentOptions */
    const componentOptions = vnode && vnode.componentOptions

    if (componentOptions) {
      /* 获取该组件节点的名称，优先获取组件的name字段，如果name不存在则获取组件的tag */
      const name = getComponentName(componentOptions)

      const { include, exclude } = this
      /* 如果name不在inlcude中或者存在于exlude中则表示不缓存，直接返回vnode */
      if (
        (include && (!name || !matches(include, name))) ||
        // excluded
        (exclude && name && matches(exclude, name))
      ) {
        return vnode
      }

      const { cache, keys } = this
      const key = vnode.key == null
        // same constructor may get registered as different local components
        // so cid alone is not enough (##3269)
        ? componentOptions.Ctor.cid + (componentOptions.tag ? `::${componentOptions.tag}` : '')
        : vnode.key
      if (cache[key]) {
        vnode.componentInstance = cache[key].componentInstance
        // make current key freshest
        remove(keys, key)
        keys.push(key)
      } else {
        cache[key] = vnode
        keys.push(key)
        // prune oldest entry
        if (this.max && keys.length > parseInt(this.max)) {
          pruneCacheEntry(cache, keys[0], keys, this._vnode)
        }
      }

      vnode.data.keepAlive = true
    }
    return vnode || (slot && slot[0])
  }
}
```

#  生命周期钩子
组件一旦被 <keep-alive> 缓存，那么再次渲染的时候就不会执行 created、mounted 等钩子函数，但是我们很多业务场景都是希望在我们被缓存的组件再次被渲染的时候做一些事情，好在Vue 提供了 activated和deactivated 两个钩子函数，它的执行时机是 <keep-alive> 包裹的组件激活时调用和停用时调用，下面我们就通过一个简单的例子来演示一下这两个钩子函数，示例如下：
```js
let A = {
  template: '<div class="a">' +
  '<p>A Comp</p>' +
  '</div>',
  name: 'A',
  mounted(){
  	console.log('Comp A mounted')
  },
  activated(){
  	console.log('Comp A activated')
  },
  deactivated(){
  	console.log('Comp A deactivated')
  }
}

let B = {
  template: '<div class="b">' +
  '<p>B Comp</p>' +
  '</div>',
  name: 'B',
  mounted(){
  	console.log('Comp B mounted')
  },
  activated(){
  	console.log('Comp B activated')
  },
  deactivated(){
  	console.log('Comp B deactivated')
  }
}

let vm = new Vue({
  el: '##app',
  template: '<div>' +
  '<keep-alive>' +
  '<component :is="currentComp">' +
  '</component>' +
  '</keep-alive>' +
  '<button @click="change">switch</button>' +
  '</div>',
  data: {
    currentComp: 'A'
  },
  methods: {
    change() {
      this.currentComp = this.currentComp === 'A' ? 'B' : 'A'
    }
  },
  components: {
    A,
    B
  }
})

```
我们定义了两个组件A和B并为其绑定了钩子函数，并且在根组件中用 <keep-alive>组件包裹了一个动态组件，这个动态组件默认指向组件A，当点击switch按钮时，动态切换组件A和B。
当第一次打开页面时，组件A被挂载，执行了组件A的mounted和activated钩子函数，当点击switch按钮后，组件A停止调用，同时组件B被挂载，此时执行了组件A的deactivated和组件B的mounted和activated钩子函数。此时再点击switch按钮，组件B停止调用，组件A被再次激活，我们发现现在只执行了组件A的activated钩子函数，这就验证了文档中所说的组件一旦被 <keep-alive> 缓存，那么再次渲染的时候就不会执行 created、mounted 等钩子函数。


