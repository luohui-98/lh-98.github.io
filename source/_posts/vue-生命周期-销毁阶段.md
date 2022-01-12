---
title: vue生命周期销毁阶段
tag: vue2.0
date: 2021-8-1
---
当调用了vm.$destroy方法，Vue实例就进入了销毁阶段，该阶段所做的主要工作是将当前的Vue实例从其父级实例中删除，取消当前实例上的所有依赖追踪并且移除实例上的所有事件监听器。也就是说，当这个阶段完成之后，当前的Vue实例的整个生命流程就全部走完了



# 销毁阶段分析
当调用了实例的$destroy方法之后，当前实例就进入了销毁阶段。所以分析销毁阶段就是分析$destroy方法的内部实现。该方法的定义位于源码的src/core/instance.lifecycle.js中，如下：
```js
Vue.prototype.$destroy = function () {
  const vm: Component = this
  if (vm._isBeingDestroyed) {
    return
  }
  callHook(vm, 'beforeDestroy')
  vm._isBeingDestroyed = true
  // remove self from parent
  const parent = vm.$parent
  if (parent && !parent._isBeingDestroyed && !vm.$options.abstract) {
    remove(parent.$children, vm)
  }
  // teardown watchers
  if (vm._watcher) {
    vm._watcher.teardown()
  }
  let i = vm._watchers.length
  while (i--) {
    vm._watchers[i].teardown()
  }
  // remove reference from data ob
  // frozen object may not have observer.
  if (vm._data.__ob__) {
    vm._data.__ob__.vmCount--
  }
  // call the last hook...
  vm._isDestroyed = true
  // invoke destroy hooks on current rendered tree
  vm.__patch__(vm._vnode, null)
  // fire destroyed hook
  callHook(vm, 'destroyed')
  // turn off all instance listeners.
  vm.$off()
  // remove __vue__ reference
  if (vm.$el) {
    vm.$el.__vue__ = null
  }
  // release circular reference (##6759)
  if (vm.$vnode) {
    vm.$vnode.parent = null
  }
}
```
首先判断当前实例的_isBeingDestroyed属性是否为true，因为该属性标志着当前实例是否处于正在被销毁的状态，如果它为true，则直接return退出函数，防止反复执行销毁逻辑。
接着，触发生命周期钩子函数beforeDestroy，该钩子函数的调用标志着当前实例正式开始销毁。
首先，需要将当前的Vue实例从其父级实例中删除
如果当前实例有父级实例，同时该父级实例没有被销毁并且不是抽象组件，那么就将当前实例从其父级实例的$children属性中删除，即将自己从父级实例的子实例列表中删除。

把自己从父级实例的子实例列表中删除之后，接下来就开始将自己身上的依赖追踪和事件监听移除。

我们知道， 实例身上的依赖包含两部分：一部分是实例自身依赖其他数据，需要将实例自身从其他数据的依赖列表中删除；另一部分是实例内的数据对其他数据的依赖（如用户使用$watch创建的依赖），也需要从其他数据的依赖列表中删除实例内数据。所以删除依赖的时候需要将这两部分依赖都删除掉。
接下来移除实例内响应式数据的引用、给当前实例上添加_isDestroyed属性来表示当前实例已经被销毁，同时将实例的VNode树设置为null，
接着，触发生命周期钩子函数destroyed
最后，调用实例的vm.$off方法（关于该方法在后面介绍实例方法时会详细介绍），移除实例上的所有事件监听器。
