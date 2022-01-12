---
title: vue虚拟dom
tag: vue2.0
date: 2021-7-22
---
# 虚拟DOM简介
## 什么是虚拟dom
所谓虚拟DOM，就是用一个JS对象来描述一个DOM节点，像如下示例:
```js
<div class="a" id="b">我是内容</div>

{
  tag:'div',        // 元素标签
  attrs:{           // 属性
    class:'a',
    id:'b'
  },
  text:'我是内容',  // 文本内容
  children:[]       // 子元素
}
```
## 为什么要有虚拟DOM？
我们知道，Vue是数据驱动视图的，数据发生变化视图就要随之更新，在更新视图的时候难免要操作DOM,而操作真实DOM又是非常耗费性能的，这是因为浏览器的标准就把 DOM 设计的非常复杂，所以一个真正的 DOM 元素是非常庞大的，如下所示：
```js
let div = document.createElement('div')
let str = ''
for (const key in div) {
  str += key + ''
}
console.log(str) // 结果是非常长的
```
在页面直接操作一个dom是非常消耗新能的、所以我们再操作dom之前使用js模拟一个虚拟dom，在数据发生变化的时候通过新老虚拟dom进行比较（diff算法），从而直接更新需要改变的视图。

# Vue中的虚拟DOM

## VNode类
虚拟DOM就是用JS来描述一个真实的DOM节点。而在Vue中就存在了一个VNode类，通过这个类，我们就可以实例化出不同类型的虚拟DOM节点，源码如下
```js
// 源码位置：src/core/vdom/vnode.js

export default class VNode {
  constructor (
    tag?: string,
    data?: VNodeData,
    children?: ?Array<VNode>,
    text?: string,
    elm?: Node,
    context?: Component,
    componentOptions?: VNodeComponentOptions,
    asyncFactory?: Function
  ) {
    this.tag = tag                                /*当前节点的标签名*/
    this.data = data        /*当前节点对应的对象，包含了具体的一些数据信息，是一个VNodeData类型，可以参考VNodeData类型中的数据信息*/
    this.children = children  /*当前节点的子节点，是一个数组*/
    this.text = text     /*当前节点的文本*/
    this.elm = elm       /*当前虚拟节点对应的真实dom节点*/
    this.ns = undefined            /*当前节点的名字空间*/
    this.context = context          /*当前组件节点对应的Vue实例*/
    this.fnContext = undefined       /*函数式组件对应的Vue实例*/
    this.fnOptions = undefined
    this.fnScopeId = undefined
    this.key = data && data.key           /*节点的key属性，被当作节点的标志，用以优化*/
    this.componentOptions = componentOptions   /*组件的option选项*/
    this.componentInstance = undefined       /*当前节点对应的组件的实例*/
    this.parent = undefined           /*当前节点的父节点*/
    this.raw = false         /*简而言之就是是否为原生HTML或只是普通文本，innerHTML的时候为true，textContent的时候为false*/
    this.isStatic = false         /*静态节点标志*/
    this.isRootInsert = true      /*是否作为跟节点插入*/
    this.isComment = false             /*是否为注释节点*/
    this.isCloned = false           /*是否为克隆节点*/
    this.isOnce = false                /*是否有v-once指令*/
    this.asyncFactory = asyncFactory
    this.asyncMeta = undefined
    this.isAsyncPlaceholder = false
  }

  get child (): Component | void {
    return this.componentInstance
  }
}
```
从上面的代码中可以看出：VNode类中包含了描述一个真实DOM节点所需要的一系列属性，如tag表示节点的标签名，text表示节点中包含的文本，children表示该节点包含的子节点等。通过属性之间不同的搭配，就可以描述出各种类型的真实DOM节点。

## VNode的类型
+ 注释节点
+ 文本节点
+ 元素节点
+ 组件节点
+ 函数式组件节点
+ 克隆节点

# VNode的作用
我们在视图渲染之前，把写好的template模板先编译成VNode并缓存下来，等到数据发生变化页面需要重新渲染的时候，我们把数据发生变化后生成的VNode与前一次缓存下来的VNode进行对比，找出差异，然后有差异的VNode对应的真实DOM节点就是需要重新渲染的节点，最后根据有差异的VNode创建出真实的DOM节点再插入到视图中，最终完成一次视图更新

# 总结
本章首先介绍了虚拟DOM的一些基本概念和为什么要有虚拟DOM，其实说白了就是以JS的计算性能来换取操作真实DOM所消耗的性能。接着从源码角度我们知道了在Vue中是通过VNode类来实例化出不同类型的虚拟DOM节点，并且学习了不同类型节点生成的属性的不同，所谓不同类型的节点其本质还是一样的，都是VNode类的实例，只是在实例化时传入的属性参数不同罢了。最后探究了VNode的作用，有了数据变化前后的VNode，我们才能进行后续的DOM-Diff找出差异，最终做到只更新有差异的视图，从而达到尽可能少的操作真实DOM的目的，以节省性能
