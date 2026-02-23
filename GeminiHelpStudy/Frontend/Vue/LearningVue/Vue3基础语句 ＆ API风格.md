---
title: Vue3 基础语句与 API 风格
tags:
 - Vue3 
 - Frontend 
 - Learning 
 - API 
 - CompositionAPI
create_time: 2026-01-21
---

## 1. 基础指令 (Directives)

Vue 提供了一套特殊的属性（称为指令），以 `v-` 前缀开头，用于给 DOM 元素添加特殊行为。

### 常用指令扩展

- **`v-on` (事件绑定)**
    - **作用**：用于监听 DOM 事件，并在触发时运行一些 JavaScript 代码。
    - **简写**：`@`
    - **示例**：
        ```HTML
        <button v-on:click="increment">点击 +1</button>
        <button @click="increment">点击 +1</button>
        ```

- **`v-for` (列表渲染)**
    - **作用**：基于一个数组来渲染一个列表。
    - **语法**：`item in items`
    - **注意**：使用 `v-for` 时，**强烈建议**（在组件中是必须）提供一个唯一的 `key` 属性，以便 Vue 能跟踪每个节点的身份，从而重用和重新排序现有元素。
    - **示例**：
        ```HTML
        <ul>
          <li v-for="item in items" :key="item.id">
            {{ item.name }}
          </li>
        </ul>
        ```

- **`v-bind` (属性绑定)**
    - **作用**：动态地绑定一个或多个 attribute，或一个组件 prop 到表达式。
    - **简写**：`:`
    - **示例**：`<img :src="imageSrc">`

---

## 2. 响应式基础：ref
在 Vue 3 中，`ref` 是核心的响应式 API。
- **定义**：`ref()` 接收一个内部值，返回一个响应式的、可更改的 ref 对象。
- **特性**：此对象只有一个指向内部值的属性 `.value`。
- **作用**：当数据发生变化的时候，页面显示的内容也会跟着发生变化。
```JavaScript
import { ref } from 'vue';

let var1 = ref('test');
const message = ref('Hello Vue3'); // 声明响应式变量
```
---

## 3. 组件 API 风格对比
Vue 的组件有两种不同的风格：**选项式 API (Options API)** 和 **组合式 API (Composition API)**。
### 3.1 选项式 API (Options API)
这是 Vue 2 中的经典写法。
> [!info] 概念
> 可以用包含多个选项的对象来描述组件的逻辑，如：`data`, `methods`, `mounted` 等。
> 选项定义的属性都会暴露在函数内部的 `this` 上，它会指向当前的组件实例。


```JavaScript
<script>
export default {
	data(){  // 声明响应式对象，数据通过 return 返回
		return {
			count: 0
		}
	},
	methods: {  // 声明方法，可以通过组件实例访问
		increment: function(){
            // 选项式 API 中通过 this 访问数据
			this.count++;
		}
	},
	mounted(){ // 声明钩子函数
		console.log('Vue mounted...');
	}
}
</script>
```

### 3.2 组合式 API (Composition API) - **推荐**
这是 Vue 3 引入的基于函数的组件编写方式。

> [!info] 概念
> 通过使用函数来组织和复用组件的逻辑。它提供了一种更灵活、可组合的方式来编写组件。

#### 核心特性
1. **`<script setup>`**：这是一个标识，告诉 Vue 需要进行一些处理，让我们以更简洁的方式使用组合式 API。
2. **`ref()`**：用于声明响应式状态。
3. **`onMounted()`**：组合式 API 中的钩子方法，注册一个回调函数，在组件挂载完成后执行。

> [!danger] 重要警告：关于 `this`
> 在 Vue 的组合式 API (`<script setup>`) 中使用时，**是没有 `this` 对象的**，`this` 对象是 `undefined`。
> 切勿尝试像选项式 API 那样使用 `this.xxx` 来访问变量或方法。

#### 示例代码 (由选项式改写)
```JavaScript
<script setup>
// 1. 引入必要的函数
import { ref, onMounted } from 'vue';

// 2. 声明响应式数据
// const count = ref(0); // 声明响应式变量

const count = ref(0);

// 3. 声明函数
// 在组合式 API 中没有 this，直接定义函数即可
// 必须通过 .value 属性修改 ref 的值
function increment() {
	count.value++; // count 是 ref 对象，修改其 value 属性
}

// 4. 声明钩子函数
onMounted(() => {
	console.log('Vue mounted ...');
})
</script>

<template>
    <button @click="increment">count: {{ count }}</button>
</template>
```

---

## 4. 进阶与补充知识

### 箭头函数声明
在 `<script setup>` 中，我们常使用箭头函数来定义方法，这更加符合现代 JavaScript 的开发习惯。
```JavaScript
<script setup>
// 声明函数 - 箭头函数风格
const search = () => {
    console.log("Searching...");
}
</script>
```

### 异步操作 (Async/Await)
在处理网络请求（如使用 `axios`）时，`async` 和 `await` 是成对出现的关键语法。
> [!tip] 异步处理模式
> `async` 声明函数为异步，`await` 用于等待一个 Promise 完成（例如等待接口返回数据）。

```JavaScript
import axios from 'axios';

// 示例：异步获取数据函数
const search = async () => {
    try {
        // 等待 get 请求完成并将结果赋值给 result
        const result = await axios.get('https://api.example.com/data');
        console.log(result.data);
    } catch (error) {
        console.error("请求失败", error);
    }
}
```

---

## 5. 总结与对比

|**特性**|**选项式 API (Options API)**|**组合式 API (Composition API)**|
|---|---|---|
|**组织方式**|按照选项分类 (`data`, `methods`)|按照逻辑功能组织 (函数式)|
|**上下文**|使用 `this` 访问实例|**没有 `this`**，直接使用变量|
|**响应式数据**|`data()` 返回对象|`ref()` 或 `reactive()`|
|**复用性**|`Mixin` (容易冲突)|`Composables` (组合式函数，更清晰)|
|**适用场景**|简单组件，Vue 2 旧项目|复杂逻辑组件，Vue 3 新项目|