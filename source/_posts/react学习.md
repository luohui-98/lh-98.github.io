---
title: react学习
tag: react
date: 2021-12-20
---
# 挂在方式
```js
ReactDOM.render(
  element,
  document.getElementById('app')
);
```
通过ReactDOM.render() 挂载， 第一个参数是组件，第二个参数是挂载的根节点，需要通过原生js获取，通常只会挂载一次

# 组件使用

## 函数试组件
```js
  function Welcome(props) {
  return <h1>Hello, {props.name}</h1>;
}
```
函数式组件没有this,可以接收一个props

如果函数组件想使用state,得使用hook {useState}
```js
import React, { useState, useEffect } from 'react';

function Example() {
  const [count, setCount] = useState(0); // count定义的变量，setCount改变变量的方法，useState(0)设置变量初始值为0

  // 相当于 componentDidMount 和 componentDidUpdate:
  useEffect(() => {
    // 使用浏览器的 API 更新页面标题
    document.title = `You clicked ${count} times`;
  });

  return (
    <div>
      <p>You clicked {count} times</p>
      <button onClick={() => setCount(count + 1)}>
        Click me
      </button>
    </div>
  );
}
```

## 类组件
```js
import React from "react";
import PropsType from "prop-types";
class Welcome extends React.Component {
  // constructor(props){
  //   super(props)
  //   this.state = {
  //     age: 18
  //   }
  //   this.onChang = this.onChang.bind(this)
  // }
  static propTypes = React.PropsTypes({
    // 数据类型为字符串并且必须填
    name: PropsType.string.isRequired
    // 为函数类型
    fun: PropsType.func
  })
  static defaultProps = {
    name:'lisi'
  }
  state = {
    age: 18
  }
  onChang = () => {
    const {age} = this.state
    this.setState({
      age: age + 1
    })
    console.log('点击事件')
  }
  render() {
    return <h1 onClick={this.onChang}>Hello, {this.props.name}{this.state.age}</h1>;
  }
}
```
1. 类组件中的state与事件定义的方式有两种
   一种是在constructor里面定义，需要通过bind将事件的this改变为类的this,这种定义方式每次写一个事件就需要重新绑定一次,会比较麻烦，但是比较容易理解
   **否则事件获取不到this,因为 1、类会自己开启严格模式  2、babel也会开启严格模式**
  二种是在定义事件的时候使用赋值的方式，我们直接赋值一个箭头函数 给这个事件（因为箭头函数没有自己的this，它会将外层的this作为自己的this)

2. 设置组件的接收的参数需要引入propTypes, 设置属性需要使用静态属性**要不在类的外面设置，要是在类里面设置需要加static**
3. 改变state里面的数据需要使用this.setState() 方法
4. 子组件给父组件传递参数，父组件给子组件传递一个方法，然后子组件在方法里面传入参数，父组件可以把参数存进state
5. ref
   ```js
   class Com extends React.Component {
      changInput = () => {
        const {input,input2} = this.refs
        console.log(input,input2)
      }
      input2 = React.createRef()
      render(){
        return (
          <>
            <input ref={c => this.input = c} type="text" />
            <input ref={this.input2} type="text" />
          </>
        )
      }
   }
   ```





