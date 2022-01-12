---
title: vue模板编译
tag: vue.2.0
date: 2021-7-23
---
# 模板编译
把用户在<template></template>标签中写的类似于原生HTML的内容进行编译，把原生HTML的内容找出来，再把非原生HTML找出来，经过一系列的逻辑处理生成渲染函数，也就是render函数的这一段过程称之为模板编译过程

# 整体渲染流程
![Image text](/imgs/vue3.png)

# 模板编译内部流程
## 抽象语法树AST
抽象语法树，在计算机科学中，抽象语法树（AbstractSyntaxTree，AST），或简称语法树（Syntax tree），是源代码语法结构的一种抽象表示。它以树状的形式表现编程语言的语法结构，树上的每个节点都表示源代码中的一种结构。之所以说语法是“抽象”的，是因为这里的语法并不会表示出真实语法中出现的每个细节。比如，嵌套括号被隐含在树的结构中，并没有以节点的形式呈现；而类似于if-condition-then这样的条件跳转语句，可以使用带有两个分支的节点来表示

## 具体流程
+ 模板解析阶段：将一堆模板字符串用正则等方式解析成抽象语法树AST；
+ 优化阶段：遍历AST，找出其中的静态节点，并打上标记；
+ 代码生成阶段：将AST转换成渲染函数；

```js
export const createCompiler = createCompilerCreator(function baseCompile (
  template: string,
  options: CompilerOptions
): CompiledResult {
  // 模板解析阶段：用正则等方式解析 template 模板中的指令、class、style等数据，形成AST
  const ast = parse(template.trim(), options)
  if (options.optimize !== false) {
    // 优化阶段：遍历AST，找出其中的静态节点，并打上标记；
    optimize(ast, options)
  }
  // 代码生成阶段：将AST转换成渲染函数；
  const code = generate(ast, options)
  return {
    ast,
    render: code.render,
    staticRenderFns: code.staticRenderFns
  }
})

```
![image text](/imgs/vue4.png)

# 总结
大致分为三个阶段，分别是模板解析阶段、优化阶段和代码生成阶段
