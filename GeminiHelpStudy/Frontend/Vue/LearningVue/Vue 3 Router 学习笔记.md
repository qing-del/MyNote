---
title: Vue 3 Router 学习笔记
tags: 
  - Vue3 
  - VueRouter 
  - 前端开发 
  - 学习笔记
created_time: 2026-01-23
---

## 1. 介绍
> [!summary] 什么是 Vue Router？
> **Vue Router** 是 Vue.js 的官方路由。它为 Vue 提供富有表现力、可配置的、方便的路由管理。
> 核心定义：**路径 (Path)** 与 **组件 (Component)** 的对应关系。

- **官方文档**: [https://router.vuejs.org/zh](https://router.vuejs.org/zh)
- **作用**: 在单页面应用 (SPA) 中，通过改变 URL，在不重新请求页面的情况下，动态切换展示的组件。

---

## 2. 核心组成部分
Vue Router 主要由三个核心部分组成，它们共同协作实现路由功能：

### 2.1 Router 实例
- **描述**: 基于 `createRouter` 函数创建的路由实例。
- **作用**: 维护了整个应用的路由信息（`routes` 配置表）。
- **代码体现**: 通常在 `router/index.js` 中定义。

### 2.2 `<router-link>` 组件
- **描述**: 路由链接组件。
- **作用**: 用户点击后触发路由跳转。
- **渲染结果**: 浏览器最终将其解析为 `<a>` 标签。
- **示例**: `<router-link to='/emp'>员工管理</router-link>`

### 2.3 `<router-view>` 组件
- **描述**: 动态视图组件。
- **作用**: 用来渲染展示与当前 **路由路径** 对应的 **组件**。
- **位置**: 就像一个“占位符”，路由匹配到的组件会替换掉这个标签。

> [!tip] 工作流程
> 1. 用户点击 `<router-link>` 或地址栏 URL 改变。
> 2. `Router 实例` 监听到路径变化 (路由请求)。
> 3. `Router 实例` 在配置表中查找匹配的组件。
> 4. `Router 实例` 通知 `<router-view>` 更新，渲染对应组件。


---

## 3. 路由配置与嵌套路由 (重点)

在实际开发中（如管理后台），经常需要使用 **嵌套路由** 来实现布局（Layout）与内容页的切换。

### 3.1 嵌套路由概念
- **父级路由**: 通常是一个布局组件（Layout），包含公共部分（如侧边栏、顶部导航）和一个 `<router-view>`。
- **子级路由**: 具体的业务页面（如“员工管理”、“部门管理”），它们会在父级路由的 `<router-view>` 中渲染。

### 3.2 代码实现 (router/index.js)

以下代码展示了如何配置嵌套路由。注意 `children` 属性的使用。

```JavaScript
import { createRouter, createWebHistory } from 'vue-router'

const router = createRouter({
  // 扩展知识：createWebHistory 依赖 HTML5 History API，URL 不带 # 号
  history: createWebHistory(import.meta.env.BASE_URL),
  routes: [
    {
      path: '/',
      name: 'home',
      // 扩展知识：这里使用了路由懒加载 (Lazy Loading)
      component: () => import('../views/Login.vue')
    },
    {
      path: '/layout/', // 父级路径
      name: 'layout',
      component: () => import('../views/LayoutView.vue'),
      // 核心：在 children 中配置子路由
      children: [
        {
          path: 'helloworld', // 访问 /layout/helloworld
          name: 'helloworld',
          component: () => import('../views/HelloWorld.vue')
        },
        {
          path: 'test',
          name: 'test',
          component: () => import('../views/Test.vue')
        },
        {
          path: 'about',
          name: 'about',
          component: () => import('../views/About.vue')
        }
      ]
    }
  ]
})

export default router
```

### 3.3 布局组件实现 (views/LayoutView.vue)
父组件必须包含 `<router-view>` 才能让子路由显示出来。

```HTML
<script setup>
// 可以在这里添加布局相关的逻辑
</script>

<template>
  <div class="layout">
    <header>
      <h1>我的应用</h1>
      <nav>
        <router-link to="/layout/helloworld">HelloWorld</router-link> |
        <router-link to="/layout/test">Test</router-link> |
        <router-link to="/layout/about">About</router-link>
      </nav>
    </header>
    
    <main>
      <router-view /> 
    </main>
  </div>
</template>

<style>
/* 样式编写 */
.layout {
  padding: 20px;
}
</style>
```

---

## 4. 知识扩展 (Advanced)

### 4.1 路由懒加载 (Lazy Loading)
在上面的代码中，你看到了 `component: () => import(...)` 的写法。
- **原理**: 只有当路由被访问时，才会加载对应的组件文件。
- **好处**: 极大地减小了首屏加载包的大小，提升应用打开速度。
- **对比**:
    - 普通引入: `import Login from '../views/Login.vue'` (一开始就加载)。
    - 懒加载: `component: () => import('../views/Login.vue')` (按需加载)。

### 4.2 路由模式 (History vs Hash)
代码中使用了 `createWebHistory`。
- **Hash 模式 (`createWebHashHistory`)**:
    - URL 带有 `#` (例如 `http://localhost:5173/#/layout`).
    - 兼容性好，不需要后端配置。
- **History 模式 (`createWebHistory`)**:
    - URL 看起来像普通路径 (例如 `http://localhost:5173/layout`).
    - 更美观，**但需要后端服务器（Nginx/Apache）配置支持**，否则刷新页面可能报 404 错误。

### 4.3 路由重定向 (Redirect)
通常我们需要设置一个默认页面，例如访问 `/` 时自动跳到 `/layout`。

```JavaScript
{
  path: '/',
  redirect: '/layout/helloworld' // 访问根路径自动跳转
}
```

### 4.4 结合 Element Plus 菜单
在提供的图片 中，使用了 Element Plus 的 `<el-menu>`。
- `<el-menu>` 开启 `:router="true"` 属性后，`<el-menu-item>` 的 `index` 属性就可以直接填写入路由路径，点击菜单即可自动跳转，无需手动写 `<router-link>`。

---

## 5. 学习总结
- **Vue Router** 是连接 URL 和 组件的桥梁。
- 记住三个核心：`Router实例` (大管家), `<router-link>` (导航), `<router-view>` (展示窗口)。
- **嵌套路由** 是后台管理系统的基石，务必理解 `children` 配置和多级 `<router-view>` 的嵌套关系。