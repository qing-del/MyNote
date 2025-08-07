# Ajax 技术基础
## 目录
- [什么是Ajax](#什么是ajax)
- [Ajax使用步骤](#ajax-使用步骤)
- [Axios请求方式](#axios-请求方式)
- [核心概念](#核心概念)
  - [异步交互原理](#异步交互原理)
  - [请求参数配置](#请求参数配置)
  - [生命周期钩子](#生命周期钩子)
- [最佳实践总结](#最佳实践总结)

---

#### 什么是Ajax
● **技术定义**：Asynchronous JavaScript And XML（异步JavaScript和XML）  
● **核心作用**：  
  - 数据交换：与服务器交换数据无需页面刷新  
  - 异步交互：局部更新页面内容（如搜索联想/表单验证）  
● **技术本质**：基于XMLHttpRequest对象的异步通信机制  

#### Ajax 使用步骤
● **基础配置**：  
```html
<!-- 1. 引入Axios库 -->
<script src="axios.js"></script>

<!-- 2. 创建触发元素 -->
<button id="getBtn">获取数据</button>

<!-- 3. 发送异步请求 -->
<script>
  document.querySelector('#getBtn').addEventListener('click', () => {
    axios.get('https://api.example.com/data')
      .then(response => console.log(response.data))
  })
</script>
```

---

### Axios 请求方式
| 方法       | 使用场景          | 语法示例                               |
|------------|-------------------|----------------------------------------|
| `axios.get()` | 获取服务器数据    | `axios.get(url).then(res => {...})`    |
| `axios.post()`| 提交数据到服务器  | `axios.post(url, data).then(...)`      |
| `axios.put()` | 更新服务器资源    | `axios.put(url, data).then(...)`       |
| `axios.delete()`| 删除服务器资源   | `axios.delete(url).then(...)`          |

---

### 核心概念
#### 异步交互原理
```javascript
// 典型异步请求流程
axios.get('/api/user')
  .then(response => {
    console.log('请求成功:', response.data) // 成功回调
  })
  .catch(error => {
    console.error('请求失败:', error) // 失败处理
  })
```

#### 请求参数配置
| 参数     | 作用                  | 示例                      |
|----------|-----------------------|---------------------------|
| `url`    | 请求地址              | `'/api/users'`            |
| `method` | 请求方法(GET/POST等) | `method: 'POST'`          |
| `data`   | POST请求体数据       | `data: { name: 'John' }`  |
| `params` | GET请求URL参数       | `params: { page: 1 }`    |

#### 生命周期钩子
| 钩子函数       | 触发时机               | 典型用途                     |
|----------------|------------------------|------------------------------|
| `beforeCreate` | 请求创建前             | 初始化加载状态               |
| `created`      | 请求已创建未发送       | 添加请求头信息               |
| `beforeMount`  | 请求发送前             | 最后参数校验                 |
| `mounted`      | 响应数据到达时         | 处理响应数据                 |
| `error`        | 请求失败时             | 统一错误处理                 |

---

### 最佳实践总结
1. **请求优化**：
   - 使用拦截器统一处理请求/响应
   - 重要操作添加加载状态提示
   - 敏感操作添加防重发机制

2. **错误处理**：
```javascript
// 全局错误拦截
axios.interceptors.response.use(
  response => response,
  error => {
    alert(`请求错误: ${error.message}`)
    return Promise.reject(error)
  }
)
```

3. **安全实践**：
   - 敏感接口添加CSRF令牌
   - 用户输入参数严格过滤
   - 响应数据安全校验