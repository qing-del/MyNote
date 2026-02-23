---
title: 前端打包与 Nginx 部署指南
tags:
 - Frontend
 - Deployment 
 - Nginx 
 - Operations
 - Learning
create_time: 2026-01-23
---

## 1. 前端资源打包 (Build)
在将项目部署到服务器之前，首先需要将开发环境的代码进行编译和打包。

### 操作步骤
1.  **运行构建命令**：
    在项目的 NPM 脚本中找到 `build` 命令（通常是 `build-only` 或 `build`），点击执行。这将触发构建流程，进行类型检查并生成最终代码。
2.  **生成结果**：
    构建完成后，项目根目录下会生成一个 `dist` 文件夹。
    -   **`dist` 目录内容**：包含了项目的静态资源，通常包括 `assets` 文件夹（存放 CSS、JS、图片等）、`favicon.ico` 和入口文件 `index.html`。

> [!tip] 提示
> `dist` 目录下的文件即为最终要发布到服务器上的所有资源。

---

## 2. Nginx 基础介绍

Nginx 是前端部署中最常用的服务器软件之一。

> [!info] 概念
> Nginx 是一款轻量级的 Web 服务器/反向代理服务器及电子邮件 (IMAP/POP3) 代理服务器。
> **特点**：占有内存少，并发能力强，在各大互联网公司都有非常广泛的使用。

-   **官网地址**：[https://nginx.org/](https://nginx.org/)
-   **版本选择**：下载时通常可以看到 `Mainline version` (主力开发版) 和 `Stable version` (稳定版)，生产环境建议使用稳定版。

---

## 3. 部署流程 (Deployment)

部署的核心逻辑是将打包好的静态资源托管到 Nginx 服务器上。

### 3.1 目录结构说明
解压 Nginx 后，其主要目录结构如下：
-   **`conf`**：配置文件目录（核心文件 `nginx.conf` 在此）。
-   **`html`**：静态资源文件存放目录（默认存放 html、CSS、JS）。
-   **`logs`**：日志文件存放目录。
-   **`nginx.exe`**：可执行启动文件。

### 3.2 操作步骤
1.  **文件迁移**：
    将项目打包生成的 `dist` 目录下的**所有文件**，复制粘贴到 Nginx 安装目录下的 `html` 文件夹中。
2.  **启动服务**：
    双击运行 `nginx.exe` 文件即可启动。Nginx 服务器默认占用 **80** 端口号。
3.  **访问验证**：
    打开浏览器访问 `http://localhost:80`。如果部署成功，你将看到项目的页面（例如：Tlias智能学习辅助系统）。

---

## 4. 配置与常见问题

### 4.1 端口冲突处理
> [!danger] 注意：端口占用
> Nginx 默认占用 80 端口。如果启动失败或访问不到，很可能是 80 端口被其他程序占用了。

**排查与解决：**
1.  **检查端口占用命令**：在命令行输入 `netstat -ano | findStr 80` 查看端口占用情况。
2.  **修改端口**：如果 80 被占用，可以打开 `conf/nginx.conf` 文件，修改 `listen` 后的端口号（如改为 81 或 8080）。

### 4.2 反向代理配置 (Reverse Proxy)
为了解决前端跨域问题或正确转发 API 请求，通常需要修改 `nginx.conf` 进行反向代理配置。

- **配置示例** ：
```nginx
worker_processes 1;

events {
    worker_connections 1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;
    
    server {
        listen       80;
        server_name  localhost;

        # 静态资源处理
        location / {
            root   html;
            index  index.html index.htm;
        }

        # API 反向代理配置
        # 将 /api/ 开头的请求转发到后端服务 (例如 localhost:8080)
        location ^~ /api/ {
            rewrite ^/api/(.*)$ / $1 break; # 重写路径，去除 /api 前缀
            proxy_pass http://localhost:8080; # 转发目标地址
        }
    }
}
````

### 4.3 负载均衡
为了防止后端服务器群组中某一台的负载过高，可以在`Nginx`中设置负载均衡策略
- **配置示例**
```nginx
http {  
    ...
    upstream webservers{  
      server 127.0.0.1:8080 weight=90 ;  
      #server 127.0.0.1:8088 weight=10 ;  
    }  
  
    server {
	  ...  
    }
```

|**名称**|**说明**|
|---|---|
|轮询|默认方式|
|weight|权重方式，默认为1，权重越高被分配客户端请求越多|
|ip_hash|依据ip分配，每个客户可以固定访问一个后端服务器|
|least_conn|依据最少连接方式，把请求优先分配给连接数少的后端服务|
|url_hash|依据url分配，相同url会被分配同一台服务器|
|fair|依据响应时间，响应时间短的优先被分配|

---

## 5. 总结

|**阶段**|**核心动作**|**产物/工具**|
|---|---|---|
|**打包**|`npm run build`|`dist` 目录 (含 `index.html`, `assets`)|
|**服务器**|下载与解压|`nginx.exe`, `conf`, `html`|
|**部署**|复制文件 (`dist` -> `html`)|托管好的静态网站|
|**调试**|修改 `nginx.conf`|解决端口冲突、配置反向代理|

---