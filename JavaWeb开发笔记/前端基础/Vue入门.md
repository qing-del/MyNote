# Vue 框架基础
## 目录
- [什么是Vue](#什么是vue)
- [Vue 使用步骤](#vue-使用步骤)
- [核心指令](#核心指令)
- [JavaScript 核心概念](#javascript-核心概念)
  - [JS基础语法](#js基础语法)
  - [JSON 数据处理](#json-数据处理)
  - [DOM 操作](#dom-操作)
- [HTML & CSS 基础](#html--css-基础)
  - [Web标准三要素](#web标准三要素)
  - [CSS 样式应用](#css-样式应用)
  - [选择器优先级](#选择器优先级)
  - [盒子模型](#盒子模型)
- [最佳实践总结](#最佳实践总结)
  
#### 什么是Vue
● **渐进式框架**：用于构建用户界面的JavaScript框架[（https://cn.vuejs.org/）](https://cn.vuejs.org/)
● **框架优势**：  
  - 提供完整项目解决方案，提升开发效率  
  - 缺点：需要掌握特定使用规则  
● **核心特点**：数据驱动视图  

#### Vue 使用步骤
● **基础配置**：  
```html
<!-- 1. 引入Vue模块 -->
<script src="vue.js"></script> 

<!-- 2. 创建挂载元素 -->
<div id="app"></div>

<!-- 3. 创建应用实例 -->
<script>
  const app = Vue.createApp({
    data() { return { message: 'Hello' } }
  }).mount('#app')
</script>
```

#### 核心指令
| 指令 | 作用 | 语法示例 | 注意事项 |
|------|------|----------|----------|
| `v-for` | 列表渲染 | `<li v-for="(item,index) in items" :key="item.id">` | 必须定义`data`中的遍历数组 |
| `v-bind` | 动态绑定属性 | `:href="url"` 或 `` | 替代插值表达式绑定属性 |
| `v-if` | 条件渲染 | `<p v-if="isShow">显示内容</p>` | 通过DOM操作控制显示 |
| `v-show` | 显示/隐藏 | `<div v-show="isVisible">内容</div>` | 通过CSS的display控制 |
| `v-model` | 双向数据绑定 | `<input v-model="username">` | 表单元素专用 |

---

### JavaScript 核心概念
#### JS基础语法
```javascript
// 变量声明（避免使用var）
let count = 10   // 可变变量
const PI = 3.14  // 不可变常量

// 输出方式
console.log("调试信息")  // 控制台输出
alert("重要提示")       // 弹出警告框
```

#### JSON 数据处理
```javascript
// 对象转JSON字符串
const userStr = JSON.stringify({name: "Tom", age: 25})

// JSON字符串转对象
const userObj = JSON.parse('{"name":"Jerry","age":30}')
```

#### DOM 操作
● **获取元素**：  
```javascript
// 获取单个元素
const btn = document.querySelector('#submitBtn')

// 获取元素集合
const items = document.querySelectorAll('.list-item')
```

● **事件监听**：  
```javascript
// 推荐方式（支持多事件绑定）
btn.addEventListener('click', () => {
  console.log('按钮被点击')
})
```

---

### HTML & CSS 基础
#### Web标准三要素
| 技术 | 作用 | 示例 |
|------|------|------|
| HTML | 页面结构 | `<h1>标题</h1>` |
| CSS | 样式表现 | `h1 { color: red }` |
| JavaScript | 交互行为 | `button.addEventListener(...)` |

#### CSS 样式应用
```html
<!-- 三种引入方式 -->
<!-- 1. 行内样式 -->
<p style="color:blue">文本</p>

<!-- 2. 内部样式 -->
<style>
  p { font-size: 16px }
</style>

<!-- 3. 外部样式 -->
<link rel="stylesheet" href="style.css">
```

#### 选择器优先级
```css
/* 优先级排序 */
#id-selector { ... }    /* 最高优先级 (100) */
.class-selector { ... } /* 中等优先级 (10) */
element-selector { ... }/* 基础优先级 (1) */
```

#### 盒子模型
```css
.box {
  width: 300px;         /* 内容宽度 */
  padding: 20px;        /* 内边距 */
  border: 5px solid;    /* 边框 */
  margin: 10px;         /* 外边距 */
}
```


---

### 最佳实践总结
1. **Vue优化**：
   - 列表渲染始终使用唯一`key`
   - 频繁切换用`v-show`，条件渲染用`v-if`
   - 表单处理使用`v-model`双向绑定

2. **JS优化**：
   ```javascript
   // 事件委托示例
   document.getElementById('list').addEventListener('click', e => {
     if(e.target.classList.contains('item')) {
       console.log('点击了列表项')
     }
   })
   ```

3. **CSS优化**：
   - 避免过度嵌套选择器
   - 使用CSS变量管理主题色
   - 优先使用Flex/Grid布局