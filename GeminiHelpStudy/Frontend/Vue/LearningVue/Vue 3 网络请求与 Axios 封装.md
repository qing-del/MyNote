---
title: Vue 3 网络请求与 Axios 封装
tags:
  - Vue3 
  - Axios
  - Vite 
  - 前端工程化 
  - 跨域
created_time: 2026-01-23
---

## 1. 为什么需要封装 Axios？

> [!failure] 直接在组件中使用 Axios 的弊端
> 直接在 Vue 组件中编写 Axios 请求会带来以下问题:
> 1. **代码耦合度高**: 请求路径（URL）硬编码在组件中，一旦后端接口变更，需要修改所有组件。
> 2. **维护困难**: 难以统一管理错误处理、Token 携带等逻辑。
> 3. **数据解析繁琐**: 每次都需要手动处理 `result.data.data`。

**解决方案**: 采用 **“工具类 + API层”** 的架构。
- `utils/request.js`: 创建 Axios 实例，配置拦截器（统一处理）。
- `api/xxx.js`: 定义具体的请求函数，组件只需调用函数。

---

## 2. 跨域问题与反向代理 (重点解答)

> [!question] 你的问题：为什么一定要在 `vite.config.js` 中配置反向代理？
### 2.1 根本原因：浏览器的同源策略
浏览器为了安全，实施了 **同源策略 (Same-Origin Policy)**。当 **协议**、**域名** 或 **端口** 有任意一项不同时，就被视为跨域。
- **前端环境**: `http://localhost:5173` (Vite 启动)
- **后端接口**: `http://localhost:8080`
- **结果**: 端口不同 (5173 vs 8080)，浏览器会拦截 **响应数据**。

### 2.2 解决方案：代理 (Proxy)
配置 `vite.config.js` 是为了在 **开发环境** 下解决跨域问题。Vite 启动的本地服务器（Node.js）充当了“中间人”。

**流程解析**:
1. **前端**: 请求发给 Vite 开发服务器 (`/api/depts` -> `localhost:5173/api/depts`)。这是同源的，浏览器允许。
2. **Vite 服务器**: 识别到 `/api` 前缀，将请求 **转发 (Proxy)** 给后端 (`localhost:8080/depts`)。服务器之间没有同源策略限制。
3. **后端**: 返回数据给 Vite 服务器。
4. **Vite 服务器**: 将数据转发回前端。

### 2.3 代码实现 (`vite.config.js`)

```JavaScript
import { fileURLToPath, URL } from 'node:url'
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [vue()],
  server: {
    proxy: {
      '/api': { // 获取路径中包含 /api 的请求
        target: 'http://localhost:8080', // 后端服务实际地址
        changeOrigin: true, // 修改请求头的 host，欺骗后端
        rewrite: (path) => path.replace(/^\/api/, '') // 将 /api 替换为空字符串，因为后端接口通常不带 /api
      }
    }
  }
})
```

> [!warning] 拓展注意
> 这个代理配置 **仅在开发环境 (Development)** 有效。
> 项目打包上线 (Production) 后，`vite.config.js` 不再起作用，通常需要使用 **Nginx** 来进行反向代理配置。

---

## 3. 请求认证机制 (Token)
### 3.1 登录流程
1. 用户输入账号密码登录。
2. 登录成功，后端返回 **JWT 令牌 (Token)**。
3. 前端将 Token 存入浏览器的 `localStorage` 中。

```JavaScript
// 存储 Token
localStorage.setItem('loginUser', JSON.stringify(result.data));
// 获取 Token
const user = JSON.parse(localStorage.getItem('loginUser'));
```

### 3.2 携带令牌 (Request Interceptor)
为了让后端识别当前用户身份，后续的每一次请求都需要携带 Token。我们可以利用 Axios 的 **请求拦截器** 自动完成这一步，避免在每个请求中重复写代码。

**实现代码 (`utils/request.js`)**:
```JavaScript
// 1. 引入 axios
import axios from 'axios';

// 2. 创建实例
const instance = axios.create({
    baseURL: '/api', // 这里配合 vite 的代理
    timeout: 60000
});

// 3. 请求拦截器
instance.interceptors.request.use(
    (config) => {
        // 从 localStorage 获取令牌
        const loginUser = JSON.parse(localStorage.getItem('loginUser'));
        
        // 如果有 token，则添加到请求头中
        if (loginUser && loginUser.token) {
            config.headers.token = loginUser.token; // 这里的 key (token) 需要和后端约定
        }
        return config;
    },
    (error) => {
        return Promise.reject(error);
    }
);
```

---

## 4. 响应拦截与异常处理
当 Token 过期或非法时，后端通常返回 **401** 状态码。我们需要在 **响应拦截器** 中统一处理这种情况，强制跳转回登录页。

**实现代码 (`utils/request.js`)**:
```JavaScript
import { ElMessage } from 'element-plus'
import router from '@/router' // 引入路由实例

// 4. 响应拦截器
instance.interceptors.response.use(
    (response) => {
        // 成功回调：直接剥离一层 data，简化组件中的调用
        return response.data; 
    },
    (error) => {
        // 失败回调
        if (error.response && error.response.status === 401) {
            ElMessage.error('登录失效，请重新登录');
            router.push('/login'); // 跳转到登录页
        } else {
            ElMessage.error('接口访问异常');
        }
        return Promise.reject(error);
    }
);

export default instance;
```

---

## 5. 总结

1. **封装**: 不要直接用 Axios，要封装 `request.js`。   
2. **代理**: `vite.config.js` 中的 proxy 是为了骗过浏览器，解决开发环境的跨域问题。
3. **拦截器**:
    - **请求拦截器**: 自动在 Header 塞 Token。
    - **响应拦截器**: 统一解包数据 (`response.data`)，统一处理 401 跳转。

> [!check] 下一步行动
> 建议你尝试把这段配置复制到你的项目中，并故意把 `vite.config.js` 中的 `target` 改错，观察浏览器控制台报错（通常是 500 或 404），以此加深对代理转发的理解。