---
title: Element Plus 快速入门与核心组件
tags:
  - Vue3
  - ElementPlus
  - UI组件库
  - Frontend
create_time: 2026-01-21
---

## 1. 什么是 Element Plus

> [!info] 简介
> 
> **Element Plus** 是基于 **Vue 3**，面向设计师和开发者的 **组件库**。它是经典的 Element UI 在 Vue 3 下的升级版本，由饿了么大前端团队研发。

- **作用**：提供了一套现成的网页部件（组件），如：超链接、按钮、图片、表格、表单、分页条等，帮助开发者快速构建美观的页面。
    
- **优势**：相比原生的 HTML 标签，Element Plus 的组件样式更美观，交互更丰富（如上图所示的“完胜”对比）。
    
- **官网**：[https://element-plus.org/zh-CN/#/zh-CN](https://element-plus.org/zh-CN/#/zh-CN)
    

---

## 2. 快速入门 (Quick Start)

### 2.1 安装与引入

在使用前，需要先创建一个 Vue 3 项目，然后按照以下步骤集成：

1. **安装组件库**
    在当前工程目录下运行命令行：
    ```Bash
    npm install element-plus@2.4.4 --save
    ```
> [!tip] (注：`@2.4.4` 是示例版本，实际开发中可安装最新版)_

2. **在 `main.js` 中引入**
    需要在入口文件中引入 JS 库、CSS 样式文件，并挂载到 Vue 实例上。

    > [!tip] 扩展：中文国际化配置
    > 默认 Element Plus 的组件（如分页器、日历）是英文的。如果希望使用中文，需要在 `main.js` 中进行特殊配置。

    ```JavaScript
    import { createApp } from 'vue'
    import App from './App.vue'
    
    // 1. 引入 Element Plus
    import ElementPlus from 'element-plus'
    // 2. 引入样式文件 (必须引入，否则无样式)
    import 'element-plus/dist/index.css'
    // 3. 引入中文语言包 (扩展知识点)
    import zhCn from 'element-plus/es/locale/lang/zh-cn'
    
    const app = createApp(App)
    
    // 4. 使用组件库，并配置国际化
    app.use(ElementPlus, { locale: zhCn })
    
    app.mount('#app')
    ```

3. **使用组件**
    访问官方文档，复制需要的组件代码到 Vue 组件中，调整为项目所需的数据即可。

---

## 3. 核心组件使用详解

### 3.1 表格组件 (Table)
用于展示多条结构类似的数据。
- **使用关键点**：清楚表格绑定的数据源 `data`，以及每一列 `el-table-column` 要展示的属性信息。

#### 基础用法

```HTML
<template>
  <el-table :data="tableData" style="width: 100%">
    <el-table-column prop="date" label="Date" width="180" />
    <el-table-column prop="name" label="Name" width="180" />
    <el-table-column prop="address" label="Address" />
  </el-table>
</template>

<script lang="ts" setup>
// 项目中数据通常来自 Ajax 异步请求服务端返回
const tableData = [
  { date: '2016-05-03', name: 'Tom', address: 'No. 189, Grove St, Los Angeles' },
  // ...
]
</script>
```

#### 扩展：自定义列内容 (插槽)
当需要根据数据展示不同的内容（例如：将性别数字 `1` 转为 `男`，或添加操作按钮）时，需要使用**作用域插槽**。

> [!example] 示例：性别转换
> 在 `<el-table-column>` 内部使用 `<template #default="scope">`，其中 `scope.row` 代表当前行的数据对象。

```HTML
<el-table :data="empList" border style="width: 100%">
  <el-table-column prop="name" label="姓名" width="100"></el-table-column>
  
  <el-table-column label="性别" width="180" align="center">
    <template #default="scope">
      {{ scope.row.gender == 1 ? '男' : '女' }}
    </template>
  </el-table-column>
  
   <el-table-column label="操作">
    <template #default="scope">
       <el-button type="primary" size="small">编辑</el-button>
    </template>
  </el-table-column>
</el-table>
```

---

### 3.2 分页条组件 (Pagination)
- **场景**：当数据量过多时，使用分页操作分解数据，提升性能和用户体验。
- **常用属性**：
    - `layout`：布局（如 "prev, pager, next"）。
    - `total`：总条目数。
    - `page-size`：每页显示条数。

---

### 3.3 对话框组件 (Dialog)
- **场景**：在保留当前页面状态的情况下，告知用户并承载相关操作（如弹窗填表）。
- **关键点**：通过 `v-model` (或者 `model-value`) 绑定的 **boolean 值**，来控制 Dialog 的显示与隐藏。

```HTML
<el-dialog v-model="dialogVisible" title="提示">
    <span>这是一个对话框</span>
    <template #footer>
      <button @click="dialogVisible = false">取消</button>
      <button @click="dialogVisible = false">确定</button>
    </template>
</el-dialog>

<script setup>
import { ref } from 'vue';
const dialogVisible = ref(false); // true显示，false隐藏
</script>
```

---

### 3.4 表单组件 (Form)
- **场景**：用于数据采集（登录、注册、信息录入）。
- **关键点**：
    1. **数据采集**：使用 `v-model` 进行双向数据绑定。
    2. **数据提交**：通过事件绑定（如 `@click`）触发提交函数。

```HTML
<el-form :model="form">
    <el-form-item label="活动名称">
        <el-input v-model="form.name" />
    </el-form-item>
</el-form>
```

---

### 扩展：校验规则
- **场景** ： 假设是创建（添加）数据的请求，一般会填写一个表单用于生成带有数据的`json`
- **所需** ： 在`JavaScript`通过`rules`函数进行校验规则的编写

#### 输入内容的校验
- 这里是 JS 代码，设置校验规则
```Javascript
//表单校验
const rules = ref({
  name: [
    { required: true, message: '部门名称是必填项', trigger: 'blur'},
    { min: 2, max: 10, message: '内容长度需要在2-10位', triiger: 'blur'}
  ]
})
```
> [!info] `trigger`是校验时机属性，`blur`是失去焦点时的意思

- 这里是对应的`dialog`弹窗表单代码
```html
<el-dialog v-model="dialogFormVisible" :title="formTitle" width="500">
  <el-form :model="delp" :rules="rules"> <!-- 这里加入 rules 属性 -->
    <el-form-item label="部门名称" label-width="80px" prop="name"> 
										 <!-- 这里加入prop属性 -->
      <el-input v-model="dept.name" />
    </el-form-item>
  </el-form>
  
  <template #footer>
    <div class="dialog-footer">
      <el-button @click="dialogFormVisible = false">取消</el-button>
      <el-button type="primary" @click="save">确认</el-button>
    </div>
  </template>
</el-dialog>
```

> [!error] 注意！！！！ 
> 完成上述代码之后，仅仅只是支持校验输入的内容
> 此时即使内容不合法，依旧可以提交到服务器中

#### 提交的校验
- 编写响应对象函数
```javascript
const deptFormRef = ref();  // 响应空对象

const save = async () => {
  // 表单校验
  if(!deptFormRef.value) return; // 不存在就直接返回
						 // 在 ↓ 这里加上 async
  deptFormRef.value.validate(async (valid) => {
	if(valid) {
	  // 由于这里使用 await 所以需要在重写方法参数处补一个 async
	  const result = await addApi(dept.value);
	  //添加逻辑的代码..
	} else {
	  ElMessage.error('表单校验不通过！');
	}
  })
}
```

- 需要给表单绑定`ref`属性
```html
<el-dialog v-model="dialogFormVisible" :title="formTitle" width="500">
  <el-form :model="delp" :rules="rules" ref="deptFormRef"> <!-- 绑定ref响应对象 --> 
    <el-form-item label="部门名称" label-width="80px" prop="name"> 
      <el-input v-model="dept.name" />
    </el-form-item>
  </el-form>
  
  <template #footer>
    <div class="dialog-footer">
      <el-button @click="dialogFormVisible = false">取消</el-button>
      <el-button type="primary" @click="save">确认</el-button>
    </div>
  </template>
</el-dialog>
```

#### 重置表单校验信息
> [!warning] 提示！
> 在完成上述代码之后
> 出现输入不合法内容的表单 被关闭后
> 再次打开会依旧显示之前不合法的提示信息
> 需要在合适的地方加入清空这些提示的逻辑处理

```javascript
//重置表单的校验规则-提示信息
if (!deptFormRef.value) return;
deptFormRef.value.resetFields();
```
> [!info] 以上是模板代码 可以自行根据自己要求进行改造

![[表单校验.png]]

---

#### 扩展：watch监听
- **作用**： 侦听一个或多个响应式**数据源**，并在数据源变化时调用传入的**回调函数**
- **用法**：
	1. 导入 watch 函数
	2. 执行 watch 函数，传入要侦听的响应式数据源（ref对象）和回调函数；

- **基础用法 - 侦听变量**
```Javascript
import { ref, watch } from 'vue'
const a = ref('')

watch(a, (newVal, oldVal) => {
  console.log(`a的值位：newVal: ${newVal}, oldVal: ${oldVal}`);
})
```

- **侦听对象**
```javascript
import { ref, watch } from 'vue'
const user = ref({name: '', age: 10})
watch(user, (newVal, oldVal) => {
  console.log(`a的值：newVal: ${newVal}, oldVal: ${oldVal}`);
}, {deep: true})
```
> [!tip] 
> - `deep`属性开启之后可以侦听**对象所有属性**的变化，其**默认**是浅层侦听
> - `immediate`(bool)，是否在侦听创建时立即触发回调函数。
> - 如果要侦听对象单个变量值，可以将**侦听变量**的模板中的`a`改写为`user.value.age`即可
---