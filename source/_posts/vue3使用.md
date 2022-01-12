---
title: vue3的使用方式
tag: vue3
date: 2022-1-6
---
vue3的使用方式

# 基本使用
```js
<template>
	<div class="home">
		<!-- 父子组件传值 -->
		<son num="66" name="trist" :age="age" @baba="getSon"><a>我是插槽</a></son>
	</div>
</template>

<script>
import son from './son.vue';
import { reactive, toRefs } from 'vue';
import { useRoute, useRouter } from 'vue-router';
export default {
	name: 'home',
	components: {
		son
	},
	setup(props, ctx) {
		// 获取当前路由信息
		const route = useRoute();
		// 全局路由的实例
		const router = useRouter();
		const state = reactive({
			name: 'trist',
			age: 22,
			sex: 'boy'
		});
		// 监听子组件事件
		const getSon = val => {
			console.log(val);
		}
		return {
			...toRefs(state),
			getSon
		};
	}
};
</script>


```

# ＜script setup＞语法糖
```js
<template>
  <div>
    <div @click="log">{{ msg }}</div>
    <MyComponent />
  </div>
</template>

<script setup>
import MyComponent from './MyComponent.vue'
import { ref } from 'vue';

// 变量
const msg = ref('Hello!');//响应式数据依然需要ref

// 函数
function log() {
  console.log(msg);
}
// <script setup> 中可以使用顶层 await。结果代码会被编译成async setup()
const post = await fetch(`/api/post/1`).then(r => r.json())
</script>
```

# 类组件
跟react的类组件相似
可以跟装饰器一起使用
```js
<template>
  <div class="hello">
    <h1>{{ msg }}</h1>
    <h2 @click="(event) => setData('1', event)">{{ flag }}</h2>
    <h2>initVar:{{ initVar }}</h2>
  </div>
</template>
...
<script lang="ts">
import { Options, Vue } from "vue-class-component";
@Options({
  props: {
    msg: String,
  },
  data() {
    return {
      flag: "这是一个欢迎组件",
      initVar: 1,
    };
  },
})
export default class HelloWorld extends Vue {
  msg!: string;
  flag!: string;
  initVar!: number;
  public mounted(): void {
    console.log(this.initVar);
  }
  setData(value: string, event: PointerEvent): boolean {
    console.log(event);
    this.initVar = 1 + 1;
    this.flag = this.flag + value;
    return false;
  }
}
</script>

```
